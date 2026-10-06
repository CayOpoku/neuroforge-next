# Copy-Paste Patterns (Next.js 16 / React 19 / Prisma)

Production-shaped skeletons. Adapt names to the feature; keep files under ~200 lines. Verify each API against the installed versions — paths are written without `src/`.

## Contents
1. Prisma singleton
2. Required env helper
3. Data access function
4. Server Action contract + form
5. Route Handler (public API / webhook)
6. Paginated list (cursor)
7. `'use cache'` read + `updateTag` write
8. TanStack Query provider + prefetch
9. Zustand store provider
10. Hydration-safe patterns

---

## 1. Prisma singleton

```ts
// server/db.ts
import 'server-only'
import { PrismaClient } from '@/generated/prisma/client'   // Prisma 7: the generator's `output` path. ≤6: '@prisma/client'

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient }

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
    // Prisma 7 also needs a driver adapter here (e.g. new PrismaPg({ connectionString })) — check the project's setup
  })

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db
```

The `globalThis` guard stops dev HMR from opening a new connection pool on every reload. On serverless, pair it with a pooled connection string.

---

## 2. Required env helper

```ts
// server/env.ts
import 'server-only'

export function requiredEnv(key: string): string {
  const value = process.env[key]
  if (!value) throw new Error(`[CONFIG] Mandatory environment variable '${key}' is not set.`)
  return value
}
```

Or validate the whole env once at startup with a zod schema (`@t3-oss/env-nextjs` if the project uses it). See `smells.md` §3 for `NEXT_PUBLIC_` rules.

---

## 3. Data access function

```ts
// features/orders/server/orders.ts
import 'server-only'
import { db } from '@/server/db'
import { requireUser } from '@/server/auth'

export async function getOrderForCurrentUser(id: string) {
  const user = await requireUser()
  return db.order.findFirst({
    where: { id, organizationId: user.organizationId },    // tenant scope from the session
    select: { id: true, number: true, status: true, totalCents: true, createdAt: true },
  })
}
```

Authorise → scope → precise `select` → return a DTO. Pages, actions and Route Handlers all go through functions like this; none of them query Prisma directly.

---

## 4. Server Action contract + form

```ts
// features/orders/actions/create-order.ts
'use server'

import { updateTag } from 'next/cache'
import { requireUser } from '@/server/auth'
import { createOrderSchema } from '@/features/orders/schemas/order.schema'
import { createOrder } from '@/features/orders/server/orders'
import { validationFailure, type ActionResult } from '@/lib/action-result'

export async function createOrderAction(_prev: ActionResult<{ id: string }> | null, formData: FormData): Promise<ActionResult<{ id: string }>> {
  const user = await requireUser()
  const parsed = createOrderSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return validationFailure(parsed.error)

  const result = await createOrder(user, parsed.data)        // returns ActionResult for expected failures (quota, duplicate)
  if (!result.ok) return result

  updateTag('orders')                                         // read-your-own-writes for the user who saved
  return { ok: true, data: { id: result.data.id } }
}
```

```tsx
// features/orders/components/create-order-form.tsx
'use client'
import { useActionState } from 'react'
import { createOrderAction } from '@/features/orders/actions/create-order'
import { AppButton } from '@/components/app/app-button'

export function CreateOrderForm() {
  const [state, action, isPending] = useActionState(createOrderAction, null)
  return (
    <form action={action} className="space-y-4">
      {/* fields: name attributes match the schema keys */}
      {state && !state.ok ? <p role="alert" className="text-destructive">{state.error.message}</p> : null}
      <AppButton type="submit" loading={isPending}>Create order</AppButton>
    </form>
  )
}
```

`<form action={serverAction}>` works before hydration (progressive enhancement). Field-level errors: `backend-errors.md` §6.

---

## 5. Route Handler (public API / webhook)

```ts
// app/api/webhooks/stripe/route.ts
import { revalidateTag } from 'next/cache'
import { verifyStripeSignature } from '@/server/stripe'
import { handleStripeEvent } from '@/features/billing/server/webhooks'

export async function POST(request: Request) {
  const raw = await request.text()                                 // signature checks need the raw body
  const event = verifyStripeSignature(raw, request.headers.get('stripe-signature'))
  if (!event) return Response.json({ message: 'Invalid signature' }, { status: 400 })

  await handleStripeEvent(event)
  revalidateTag('subscriptions', 'max')                            // background job → stale-while-revalidate
  return Response.json({ received: true })
}
```

Route Handlers are for callers that aren't your own UI. Validate, authorise (signature, API key, or session), delegate to a service, return a typed `Response.json` with an explicit status. Never return a raw Prisma error.

---

## 6. Paginated list (cursor)

```ts
// features/orders/server/orders.ts
const PAGE_SIZE_MAX = 100

export async function listOrdersPage(cursor: string | undefined, limit = 25) {
  const user = await requireUser()
  const take = Math.min(limit, PAGE_SIZE_MAX)

  const rows = await db.order.findMany({
    where: { organizationId: user.organizationId },
    select: { id: true, number: true, status: true, totalCents: true, createdAt: true },
    orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
    take: take + 1,                                             // one extra row signals a next page
    ...(cursor && { cursor: { id: cursor }, skip: 1 }),
  })

  const hasMore = rows.length > take
  const items = hasMore ? rows.slice(0, take) : rows
  return { items, nextCursor: hasMore ? items.at(-1)?.id ?? null : null }
}
```

Never ship an unbounded `findMany` behind a table. Read `cursor`/`limit` from validated search params.

---

## 7. `'use cache'` read + `updateTag` write

```ts
// features/catalogue/server/products.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

export async function getProduct(slug: string) {
  'use cache'
  cacheLife('hours')
  cacheTag('products', `product:${slug}`)
  return db.product.findUnique({ where: { slug }, select: { id: true, slug: true, name: true, priceCents: true } })
}
```

```ts
// features/catalogue/actions/update-product.ts
'use server'
export async function updateProductAction(input: unknown): Promise<ActionResult> {
  const admin = await requireRole('ADMIN')
  const parsed = updateProductSchema.safeParse(input)
  if (!parsed.success) return validationFailure(parsed.error)

  const product = await updateProduct(admin, parsed.data)
  updateTag(`product:${product.slug}`)
  updateTag('products')
  return { ok: true, data: undefined }
}
```

Requires `cacheComponents: true`. On older projects use the project's model (`data-fetching.md` §2).

---

## 8. TanStack Query provider + prefetch

See `data-fetching.md` §4 for `getQueryClient()`, `Providers`, `queryOptions` and the `HydrationBoundary` prefetch — copy from there rather than re-deriving.

Mount `<Providers>` in the **root layout** around `{children}`; the provider is a Client Component, the layout stays a Server Component.

---

## 9. Zustand store provider

See `data-fetching.md` §5 — `createStore` from `zustand/vanilla`, a context provider that creates one store per tree with `useState(() => …)`, and a selector hook that throws outside the provider.

---

## 10. Hydration-safe patterns

```tsx
// Wrong — localStorage during render: crashes on the server, mismatches on the client
const theme = localStorage.getItem('theme') ?? 'light'
// Right — read a cookie on the server and pass it down (or next-themes, which handles the script + suppressHydrationWarning)
const theme = (await cookies()).get('theme')?.value ?? 'light'

// Wrong — time or randomness rendered on both sides
<p>{new Date().toLocaleTimeString()}</p>
// Right — render on the server with an explicit locale/timeZone, or render after mount in a small client component
<p>{formatTime(createdAt, { locale: 'en-GB', timeZone: user.timeZone })}</p>

// Wrong — a browser-only library imported at module scope in a client file
import Map from 'browser-only-map'
// Right — load it client-side only
const Map = dynamic(() => import('@/components/map'), { ssr: false, loading: () => <MapSkeleton /> })
```

`dynamic(..., { ssr: false })` must be called from a **Client Component** in the App Router. For anything else reactive, check `rendering.md` before writing `useEffect`.
