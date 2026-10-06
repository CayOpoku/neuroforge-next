# Server Contracts: Input Validation & Error Handling

Validation and errors are one subject. An error message is only trustworthy if the boundary that produced it actually checked its input.

---

## 1. Validate every boundary with a schema

A Server Action is a **public POST endpoint**. Anyone can call it with any payload — the TypeScript signature is a hope, not a check. The same goes for a Route Handler's `request.json()`, `params`, and `searchParams`. Every boundary parses its input with a schema first.

```ts
// lib/schemas/order.schema.ts — shared so the form and the action agree
import { z } from 'zod'

export const createOrderSchema = z.object({
  email: z.string().email(),
  quantity: z.coerce.number().int().positive().max(100),
  note: z.string().max(500).optional(),
})

export type CreateOrderInput = z.infer<typeof createOrderSchema>
```

```ts
// features/orders/actions/create-order.ts
'use server'

import { createOrderSchema } from '@/lib/schemas/order.schema'
import { requireUser } from '@/server/auth'
import { createOrder } from '@/features/orders/server/orders'
import type { ActionResult } from '@/lib/action-result'

export async function createOrderAction(_prev: ActionResult | null, formData: FormData): Promise<ActionResult<{ id: string }>> {
  const user = await requireUser()                                   // auth inside the action, always
  const parsed = createOrderSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return validationFailure(parsed.error)          // §1 shape below

  const order = await createOrder({ ...parsed.data, userId: user.id })
  return { ok: true, data: { id: order.id } }
}
```

- Put schemas in `lib/schemas/` (or `features/<x>/schemas/`) so the client form and the server boundary validate against **one definition**. `z.infer` replaces the hand-written type.
- Shape validation failures once, in a helper, so every form renders field errors the same way:

```ts
// lib/action-result.ts
import type { ZodError } from 'zod'

export type FieldError = { path: string; message: string }

export type ActionResult<T = undefined> =
  | { ok: true; data: T }
  | { ok: false; error: { message: string; code?: string; fields?: FieldError[] } }

export function validationFailure(error: ZodError): ActionResult<never> {
  return {
    ok: false,
    error: {
      message: 'Validation failed',
      code: 'VALIDATION',
      fields: error.issues.map((i) => ({ path: i.path.join('.'), message: i.message })),
    },
  }
}
```

- Route Handlers: `const body = schema.parse(await request.json())` inside a `try`, returning `Response.json({ message, fields }, { status: 400 })` on a `ZodError`.
- Never trust an id from the client to scope a query. See `auth.md`.
- valibot is a drop-in alternative if bundle size matters — the rule is a schema at the boundary, not a specific library.

---

## 2. Expected errors are returned; unexpected errors are thrown

This is the Next-specific rule that decides whether the user ever sees the real message.

**In production, Next.js redacts the message of any error thrown from a Server Component or Server Action.** The client receives a generic message and a `digest` — the original text exists only in the server log. So:

| Kind | Example | How it leaves the server |
| :--- | :--- | :--- |
| **Expected** — the user can act on it | validation failure, "Email already registered", "Tenant quota exceeded", 404 on a resource | **Returned** as a value: `{ ok: false, error: { message } }` from the action; a JSON body + status from a Route Handler; `notFound()` from a page |
| **Unexpected** — a bug or an outage | DB down, null dereference, upstream 500 | **Thrown.** Caught by the nearest `error.tsx`; logged server-side with the digest |

Throwing an expected error from a Server Action is the Next.js version of a silent fallback: in development the message shows, in production the user sees "An error occurred in the Server Components render" and nobody notices until it ships. **Return what the user can fix; throw what they cannot.**

---

## 3. Zero hardcoded API error messages

- **The frontend never invents an API error string.** No `"An unexpected error occurred"`, `"Failed to fetch data"`, or `"Operation failed"` written into a component, hook, or form handler.
- **All user-facing API errors originate from the primary backend** — the Server Action / service layer, or the external API (FastAPI, NestJS, Express, Strapi) behind it.
- **A Route Handler or Server Action proxying an external API forwards the upstream message intact.** A BFF layer must not replace a real backend error with a synthetic generic one.
- **Audit rule:** during analysis, flag every hardcoded string inside a `catch` block, an `onError`, or a returned `{ ok: false }` that discards the upstream message.

### The one exception: transport-level failures

If the request never reached the backend, there is no backend message to render, and `String(error)` would show the user `TypeError: Failed to fetch`. These four cases are legitimately frontend-owned and belong in `getErrorMessage`, nowhere else:

| Case | Message |
| :--- | :--- |
| Network failure / offline | "Could not reach the server. Check your connection and try again." |
| Timeout / aborted | "The request timed out. Please try again." |
| 5xx with an empty body | "Something went wrong on our end. Please try again shortly." |
| Genuinely unparseable payload | "An unexpected error occurred." |

Everything else must come from the response body or the action result.

---

## 4. `fetch` does not throw on 4xx — the `ApiError` + `getErrorMessage` pair

`fetch` resolves on a 404 or 422. Code that does `await fetch(url).then(r => r.json())` treats an error envelope as data. Every client-side call goes through one helper that turns a non-OK response into a typed error carrying the parsed body:

```ts
// lib/api.ts
export class ApiError extends Error {
  constructor(public status: number, public body: unknown) {
    super(`HTTP ${status}`)
  }
}

export async function fetchJson<T>(input: RequestInfo, init?: RequestInit): Promise<T> {
  const res = await fetch(input, init)
  const text = await res.text()
  const body: unknown = text ? safeJson(text) : null
  if (!res.ok) throw new ApiError(res.status, body)
  return body as T
}

function safeJson(text: string): unknown {
  try { return JSON.parse(text) } catch { return text }
}
```

```ts
// lib/error.ts
import { ApiError } from '@/lib/api'

interface BackendErrorPayload {
  message?: string | string[]
  detail?: string | Array<{ msg?: string; loc?: string[] }>
  error?: string | { message?: string }
}

const TRANSPORT_FALLBACK = 'Could not reach the server. Check your connection and try again.'
const TIMEOUT_FALLBACK = 'The request timed out. Please try again.'
const SERVER_FALLBACK = 'Something went wrong on our end. Please try again shortly.'
const UNKNOWN_FALLBACK = 'An unexpected error occurred.'

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null
}

function fromPayload(payload: unknown): string | undefined {
  if (typeof payload === 'string' && payload) return payload
  if (!isRecord(payload)) return undefined
  const p = payload as BackendErrorPayload

  // FastAPI: { detail: '...' } or { detail: [{ msg, loc }] }
  if (typeof p.detail === 'string') return p.detail
  if (Array.isArray(p.detail)) {
    const messages = p.detail.map((d) => d.msg).filter(Boolean)
    if (messages.length) return messages.join(', ')
  }
  // NestJS / Express: { message } or { message: [] }
  if (typeof p.message === 'string' && p.message) return p.message
  if (Array.isArray(p.message) && p.message.length) return p.message.join(', ')
  // Strapi: { error: { message } }; generic: { error: '...' }
  if (isRecord(p.error) && typeof p.error.message === 'string') return p.error.message
  if (typeof p.error === 'string') return p.error
  return undefined
}

export function getErrorMessage(error: unknown): string {
  if (!error) return UNKNOWN_FALLBACK
  if (typeof error === 'string') return error

  // ActionResult failure passed straight in
  if (isRecord(error) && error.ok === false && isRecord(error.error)) {
    const m = (error.error as { message?: unknown }).message
    if (typeof m === 'string' && m) return m
  }

  if (error instanceof ApiError) {
    const fromBody = fromPayload(error.body)
    if (fromBody) return fromBody
    return error.status >= 500 ? SERVER_FALLBACK : UNKNOWN_FALLBACK
  }

  // Transport-level: the request never produced a backend payload.
  if (error instanceof DOMException && (error.name === 'AbortError' || error.name === 'TimeoutError')) return TIMEOUT_FALLBACK
  if (error instanceof TypeError) return TRANSPORT_FALLBACK

  if (error instanceof Error && error.message) return error.message
  return UNKNOWN_FALLBACK
}
```

Field-level validation errors (`error.fields` from §1) are rendered next to their inputs, not flattened into a toast. `getErrorMessage` is for the summary line.

---

## 5. The message is terminal — never `||` a fallback onto it

**The single worst thing you can do to this system is put a default after the utility that reads the backend.**

```ts
// ✗ All of these are banned
toast.error(getErrorMessage(error) || 'Something went wrong')
toast.error(getErrorMessage(error) ?? 'Failed to save changes')
const message = getErrorMessage(error) || DEFAULT_ERROR
catch (error) { toast.error('Could not load orders') }
catch { /* ignore */ }
if (!result.ok) return { ok: false, error: { message: 'Save failed' } }   // discards the real one
```

### Why this is a hard rule, not a style preference

`getErrorMessage` returns a non-empty string in **every** branch — §4 ends in `UNKNOWN_FALLBACK`. So the `||` arm is either dead code, or it is live and you have just proved the utility has a hole. Both cases are resolved in `lib/error.ts`, never at the call site.

What the `||` actually does is fire the day the backend changes shape — a new error envelope, a renamed field, a proxy that wraps the payload. On that day the real message (*"Order total must be positive"*, *"Tenant quota exceeded"*, *"relation orders.user_id does not exist"*) is silently replaced with a reassuring generic, the UI looks like it is working as designed, and nobody — user or developer — learns that the backend broke. **A silent sweep is more expensive than a crash**: the crash is fixed the same afternoon; the generic string survives to production and gets reported months later as "it sometimes doesn't save".

### The rules

- **`getErrorMessage(error)` is terminal.** Its return value goes straight into the toast, alert or field. No `||`, no `??`, no falsy ternary, no `.trim() ||`, no wrapping helper that supplies a default.
- **Same for every sibling utility** — `getFieldErrors`, `parseApiError`, `useApiError`, anything whose job is to extract backend truth.
- **Fallbacks live inside the utility**, as the four transport constants of §3–§4, and nowhere else. If a real error shape falls through to `UNKNOWN_FALLBACK`, add that shape's branch to `getErrorMessage` and a test for it (`testing.md`).
- **Never swallow.** Every `catch` renders the message, returns it as an `ActionResult`, or rethrows. `catch {}`, `catch { return null }`, `catch { return [] }`, `.catch(() => undefined)` and a bare `console.error` with no UI are the same bug wearing different hats.
- **Keep the raw shape reachable while developing:**

```ts
catch (error) {
  if (process.env.NODE_ENV === 'development') console.error('[orders] create failed', error)
  toast.error(getErrorMessage(error))
}
```

- **A generic string on screen is a finding.** If the user sees "An unexpected error occurred", treat it as an unhandled backend shape and go read the actual response.

### Audit

Grep before you claim a codebase is clean. Every hit is a finding:

```bash
grep -rnE "getErrorMessage\(.*\)\s*(\|\||\?\?)" app/ features/ lib/ components/
grep -rnE "catch\s*(\(.*\))?\s*\{\s*\}" app/ features/ lib/ server/
grep -rn "catch" app/ features/ | grep -iE "'(Something went wrong|Failed to|An error|Unable to)"
```

---

## 6. Rendering errors

**Server Action with a form** — `useActionState` keeps the result, including field errors, without client fetch plumbing:

```tsx
'use client'
import { useActionState } from 'react'
import { createOrderAction } from '@/features/orders/actions/create-order'

export function CreateOrderForm() {
  const [state, formAction, isPending] = useActionState(createOrderAction, null)
  const fieldError = (name: string) => state && !state.ok ? state.error.fields?.find((f) => f.path === name)?.message : undefined

  return (
    <form action={formAction}>
      <input name="email" aria-invalid={!!fieldError('email')} aria-describedby="email-error" />
      {fieldError('email') ? <p id="email-error" role="alert">{fieldError('email')}</p> : null}
      {state && !state.ok && !state.error.fields ? <p role="alert">{state.error.message}</p> : null}
      <button disabled={isPending}>Create order</button>
    </form>
  )
}
```

**TanStack Query mutation** — the message goes in `onError` (`data-fetching.md`):

```ts
onError: (error) => toast.error(getErrorMessage(error)),
```

---

## 7. No false fallbacks

- **The smell:** defaulting UI state when a call fails — showing `status = 'Pending'` or `role = 'User'` because the fetch errored. This hides a system fault and shows the user a value that is not true.
- **The rule:** render the failure — an alert or a toast carrying the backend message, so both the user and the developer see what happened.
- `initialData: []` / `?? []` for a list *shape* is fine. `?? { status: 'Pending' }` is not.
- Distinguish the four states in every data-bound view: loading, empty, error, success. Collapsing "error" into "empty" is the same bug wearing a different hat.
