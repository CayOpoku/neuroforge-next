# Auth, Proxy & Route Protection

---

## 1. Two layers, only one of them is security

| Layer | Runs | Purpose |
| :--- | :--- | :--- |
| `proxy.ts` (Next 16) / `middleware.ts` (≤15) | Before routing, on matched requests | **Optimistic** checks only: redirect a visitor with no session cookie to `/login`, locale routing, rewrites, headers |
| The **data access layer** (DAL) — `server/auth.ts` + `features/*/server/*` | Inside every Server Component read, Server Action and Route Handler | **The real gate:** verify the session, check permission, scope the query |

**The request interceptor is not your authorisation boundary.** It can be skipped or bypassed — CVE-2025-29927 let attackers skip middleware entirely with a single header, and any route the `matcher` misses is unprotected by definition. Treat it as "don't show the user a broken page"; the DAL is what keeps data safe.

Equally: **a layout is not a gate.** Layouts don't re-render on client navigation between their child routes, so a session check in `layout.tsx` does not run for every page it wraps. Check in the page's data access, not the layout.

---

## 2. The data access layer

```ts
// server/auth.ts
import 'server-only'
import { cache } from 'react'
import { redirect } from 'next/navigation'
import { getSession } from '@/server/session'          // the project's auth library (Better Auth, Auth.js, Clerk, Lucia-style)
import type { SessionUser } from '@/types/auth.types'

// cache(): one session lookup per request, however many components ask
export const getCurrentUser = cache(async (): Promise<SessionUser | null> => {
  const session = await getSession()
  return session?.user ?? null
})

export async function requireUser(): Promise<SessionUser> {
  const user = await getCurrentUser()
  if (!user) redirect('/login')
  return user
}

export async function requireRole(role: SessionUser['role']): Promise<SessionUser> {
  const user = await requireUser()
  if (user.role !== role) throw new ForbiddenError()    // map to 403 in Route Handlers; notFound() for pages if existence must not leak
  return user
}
```

```ts
// features/orders/server/orders.ts
import 'server-only'
import { db } from '@/server/db'
import { requireUser } from '@/server/auth'

export async function listOrdersForCurrentUser() {
  const user = await requireUser()
  return db.order.findMany({
    where: { userId: user.id },                         // scoped by the session, never by a client id
    select: { id: true, number: true, status: true, totalCents: true, createdAt: true },
    orderBy: { createdAt: 'desc' },
    take: 50,
  })
}
```

- Every function that reads or writes user-owned data takes the user **from the session inside the function**, not as an argument a caller could forge.
- `redirect()` and `notFound()` throw — don't wrap them in a `try/catch` that swallows them.
- Return DTOs with only the fields the caller needs. Never return a user row with `passwordHash` "because the component only reads `name`".

---

## 3. Server Actions are public endpoints

A Server Action is reachable by anyone who can send a POST to your site — whether or not the button that calls it is rendered for them. Hiding the button is UX; it is not access control.

```ts
'use server'
export async function deleteInvoiceAction(id: string): Promise<ActionResult> {
  const user = await requireUser()                                   // 1. who
  const parsed = z.string().uuid().safeParse(id)                     // 2. what
  if (!parsed.success) return validationFailure(parsed.error)

  const deleted = await db.invoice.deleteMany({                      // 3. scoped: only theirs
    where: { id: parsed.data, organizationId: user.organizationId },
  })
  if (deleted.count === 0) return { ok: false, error: { message: 'Invoice not found', code: 'NOT_FOUND' } }

  updateTag('invoices')
  return { ok: true, data: undefined }
}
```

- **Every action authorises itself.** Not "the page already checked", not "proxy protects `/app/*`".
- **Scope by the session, not by a client-supplied id.** `deleteInvoice(id)` without the tenant in the `where` is an IDOR — the single most common auth bug in a SaaS dashboard.
- **Don't close over secrets.** Values captured by an inline action defined in a Server Component are encrypted and sent to the client; keep secrets in server modules, not in closures.
- Rate-limit actions that send mail, create accounts or cost money.

---

## 4. The interceptor — optimistic redirects only

```ts
// proxy.ts (Next 16) — middleware.ts with `export function middleware` on ≤15
import { NextResponse, type NextRequest } from 'next/server'

const PROTECTED = ['/app', '/settings']

export function proxy(request: NextRequest) {
  const isProtected = PROTECTED.some((p) => request.nextUrl.pathname.startsWith(p))
  const hasSession = request.cookies.has('session')     // presence only — no DB call, no verification here

  if (isProtected && !hasSession) {
    const url = new URL('/login', request.url)
    url.searchParams.set('redirect', request.nextUrl.pathname)
    return NextResponse.redirect(url)
  }
  return NextResponse.next()
}

export const config = { matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'] }
```

- **Cookie presence, not verification.** No database lookup in the interceptor — it runs on every matched request, including prefetches, and Next recommends against relying on shared modules or globals there.
- **A matcher exclusion also skips Server Actions.** Server Actions are POSTs to the page that uses them, so a matcher that excludes a path skips every action called from it — and moving an action to another route can silently remove proxy coverage. One more reason the check lives inside the action.
- **Validate the `redirect` param** on the login side: only same-origin relative paths, or it's an open redirect.
- **Export:** a single function, either `export function proxy` or `export default function proxy`. The `matcher` must be a constant (no variables). `proxy.ts` always runs on Node.js — setting `runtime` there throws. `middleware.ts` still works (Edge) but is deprecated; migrate with `npx @next/codemod@canary middleware-to-proxy .` — suggest it, don't run it.

---

## 5. Multi-tenancy

Every query on a tenant-owned model carries the tenant from the session:

```ts
const user = await requireUser()

const invoice = await db.invoice.findFirst({
  where: { id: invoiceId, organizationId: user.organizationId },    // tenant scope is part of the WHERE, always
})
if (!invoice) notFound()
```

Return **404, not 403**, for a resource in another tenant — a 403 confirms the record exists. Flag any tenant-scoped query missing its tenant filter as a critical finding.

---

## 6. Session handling rules

- Sessions live in **httpOnly, Secure, SameSite** cookies. Never persist a token in `localStorage` — it is XSS-readable and invisible to the server render.
- One client-side entry point for "who am I" (the auth library's hook or a context fed by the server). Components don't read the cookie themselves.
- Pass the user from a Server Component to the client tree as a **minimal DTO** (id, name, avatar, role) — never the whole session object.
- On a 401 from the API layer, clear the session and redirect once.
- Never log tokens, password hashes or full session objects. Log the user id.
- Auth env vars (`AUTH_SECRET`, provider keys) are mandatory: fail loudly at startup if missing (`smells.md` §3), and **never** `NEXT_PUBLIC_`.
