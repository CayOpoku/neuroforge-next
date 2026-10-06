# Type Safety, Type Placement & Per-File-Type Rules

---

## 1. `any` vs `unknown`

- **`any` is banned.** It disables checking and silently propagates.
- **`unknown` is the correct tool** for values whose shape you do not control — caught errors, third-party payloads, `JSON.parse` output, `request.json()`, `FormData` values. Narrow it before use, ideally with the boundary schema (`backend-errors.md` §1).

```ts
function readMessage(error: unknown): string {
  if (error instanceof Error) return error.message
  if (typeof error === 'object' && error !== null && 'message' in error) {
    return String((error as { message: unknown }).message)
  }
  return 'Unknown error'
}
```

- Never "fix" a typecheck error by widening to `any` or adding a blind `as`. Find why the type is wrong. A cast is a claim you must be able to justify in one sentence.
- A typed Server Action signature is **not validation** — the action is a public endpoint and receives whatever the caller sends. Parse with a schema.
- `as` on `await request.json()` is the same lie. Parse it.

---

## 2. Type placement (non-negotiable)

- **≤ 5 lines:** may live inline in the file that uses it.
- **> 5 lines:** MUST move to a dedicated `.types.ts` file. Never inline in a component, hook, Server Action, or Route Handler.
- **Where it lives depends on who uses it:**
  - one feature → `features/<feature>/types/<name>.types.ts`
  - shared across features → `types/<name>.types.ts`
  - server-only (DB rows, internal service shapes) → next to the server module, never imported by client files
- **Schemas are types too.** A zod schema in `lib/schemas/` plus `z.infer` replaces a hand-written request type — never maintain both.
- **Never create a new types file when a relevant one exists.** Extend it.
- Always import types explicitly: `import type { Order } from '@/features/orders/types/order.types'`.
- Derive from Prisma rather than hand-rewriting:

```ts
// features/orders/types/order.types.ts
import type { Prisma } from '@/generated/prisma/client'   // Prisma 7 generated path; '@prisma/client' on ≤6

export type OrderListItem = Prisma.OrderGetPayload<{
  select: { id: true; number: true; status: true; totalCents: true; createdAt: true }
}>
```

Derive from the source of truth (`Pick`, `Omit`, `Prisma.XGetPayload<…>`) so the schema stays the single definition. **Serialisation caveat:** `Decimal` and `BigInt` fields don't cross the server → client boundary as-is — convert them in the data layer and type the converted shape.

---

## 3. Next.js-generated types

Next generates route types during `next dev` / `next build` (and `next typegen` on versions that have it). Prefer them over hand-typing props:

```tsx
// app/(app)/orders/[id]/page.tsx
export default async function OrderPage(props: PageProps<'/orders/[id]'>) {
  const { id } = await props.params          // params and searchParams are Promises in Next 15+
  // …
}
```

- `PageProps<'/route'>`, `LayoutProps<'/route'>` and `RouteContext<'/api/route'>` are global helpers on versions that ship them — check the installed version; on older ones, type `params: Promise<{ id: string }>` by hand.
- **`params` is still untrusted.** It's typed as `string`, not as a valid UUID — validate before querying (`layouts-routing.md` §3).
- `typedRoutes: true` makes `<Link href>` and `router.push` type-checked against real routes. Recommend it for apps with many internal links.

---

## 4. Type fix workflow

1. Run `npx tsc --noEmit` (or the project's `typecheck` script); capture the full output. Ask before running it (`debugging.md` §2).
2. Group errors by root cause, not by file.
3. For each group: fix the cause — a missing `select` field, a wrong generic, an unvalidated boundary, an un-awaited `params` — not a lazy annotation.
4. If the root cause is unclear, **pause and ask**.
5. Create or extend `.types.ts` files as needed.
6. Re-run to confirm zero errors. Report the before/after count.

Note `next lint` was removed in Next 16 — lint runs through `eslint` directly (the project's `lint` script), and `next build` no longer lints.

---

## 5. Rules by file type

### Hooks (`hooks/` or `features/<x>/hooks/`)
- Name starts with `use` — `useOrderFilters.ts`. A function that calls no hooks is not a hook: it goes in `lib/` without the prefix.
- Client-only by nature; the consuming component is `'use client'`.
- One job per hook. `useOrdersList` + `useCreateOrder`, never one `useOrders` doing reads, writes and UI state.
- Pure helpers live **outside** the hook function.
- Every listener, timer and subscription is cleaned up in the effect's return (`rendering.md` §6).
- Return a stable, destructuring-friendly shape.

### Server Components (default)
- May be `async`; read data through the data layer. No hooks, no state, no browser APIs.
- Select only the fields the UI renders before passing anything to a Client Component.

### Client Components (`'use client'`)
- Props typed with a `type XProps = { … }`; no `React.FC` needed.
- UI and interaction only; data loading and business rules stay on the server or in hooks.
- Writes call a Server Action — never a hand-rolled `fetch` to an internal endpoint.

### Server Actions (`'use server'`)
- Authenticate and authorise **inside** the action (`auth.md`), then validate with a schema, then call a service, then invalidate (`data-fetching.md` §2).
- Return a typed `ActionResult` for expected failures; throw only for unexpected ones (`backend-errors.md` §2).
- Thin: over ~40 lines means business logic that belongs in `features/<x>/server/`.

### Route Handlers (`route.ts`)
- For public APIs, webhooks, third-party callbacks and file responses — not for mutations the app itself performs.
- Validate → authorise → service → typed `Response.json(...)` with an explicit status.
- Never return a raw Prisma error; map it to a status and a message.

### Prisma
- Singleton (`patterns.md` §1). Precise `select`. Paginate every list. Scope every query by the session (`auth.md`).
- Schema-first; `prisma generate` in `postinstall` (or the build), and import from the generated client path the project's generator declares.
