# Data Fetching, Caching & State Ownership

The single highest-leverage area in a Next.js SaaS app. Read before writing any fetch, Server Action, `'use cache'`, `useQuery`, or store.

Check the installed `next` version and whether `cacheComponents` is on before applying §2 — the caching model changed in Next 16, and the two models do not mix.

---

## 1. Pick the right tool

| Situation | Use |
| :--- | :--- |
| Data the page needs to render, owned by the server | **Server Component** — `await` it directly (Prisma via the data layer, or `fetch`) |
| Read-mostly data shared across requests (catalogue, plans, CMS pages) | Server Component + **`'use cache'`** with `cacheLife` and `cacheTag` |
| Any write the app itself performs | **Server Action** (`'use server'`) — validate, authorise, mutate, then invalidate |
| Client-driven reads: infinite scroll, polling, optimistic lists, refetch-on-focus, data shared by several client components | **TanStack Query** `useQuery` / `useInfiniteQuery`, prefetched on the server where it matters |
| A write with optimistic UI and client cache invalidation | TanStack Query `useMutation` calling a Server Action, or `useOptimistic` for a single form |
| Filters, tabs, sort, page number, search | **URL search params** (`nuqs`) — not React state, not a store |
| Small client UI state (sidebar open, wizard step, selected rows) | `useState`, lifted or in context |
| Global client UI state with behaviour (multi-step builder, cart before checkout, editor state) | **Zustand** store (§5) |
| Public API, webhook, third-party callback, file download | **Route Handler** (`route.ts`) |
| Typed RPC across a large app | tRPC — only if the project already uses it |

**Server data does not belong in a Zustand store.** Copying a server response into a store creates a second cache nobody invalidates. The server (or TanStack Query) owns it; the store owns UI state.

**Never fetch in `useEffect`** (`rendering.md` §3). Never call your own Route Handler from a Server Component — call the function the handler would call.

---

## 2. Caching — the Next.js 16 model (`cacheComponents: true`)

**Nothing is cached by default.** Every page and every fetch runs per request unless you opt in. Caching is a decision you make and can name.

```ts
// next.config.ts
const nextConfig: NextConfig = { cacheComponents: true }
```

```ts
// features/catalogue/server/plans.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'
import { db } from '@/server/db'

export async function getPublicPlans() {
  'use cache'
  cacheLife('hours')       // named profile: seconds | minutes | hours | days | weeks | max, or a custom one
  cacheTag('plans')        // what invalidation will target

  return db.plan.findMany({ select: { id: true, name: true, priceCents: true }, orderBy: { priceCents: 'asc' } })
}
```

- **`'use cache'`** goes at the top of an async function, a component, or a file (every export then becomes cached and must be async). The cache key is built from the arguments and closed-over values — keep them **small and serialisable** (no class instances, functions, or `URL` objects), and never pass a whole request object.
- **Always call `cacheLife` explicitly** in every cached scope. Without it the `default` profile applies silently, and nesting a short-lived cache inside a scope with no explicit `cacheLife` fails the build.
- **No request data inside a cached scope — and the rule follows the call stack.** `cookies()`, `headers()` and `searchParams` cannot be read inside `'use cache'` *or in any helper it calls*. Read them outside and pass the specific value in as an argument (each distinct value is its own cache entry). On a dynamic route this mistake can pass `next build` and only fail under `next start` — trace the helpers, don't trust a green build. Per-user data is usually not a `'use cache'` candidate at all.
- **The static shell.** With `cacheComponents`, `'use cache'` output plus `<Suspense>` fallbacks form the prerendered shell (Partial Prerendering); everything else streams. Awaiting uncached data, `cookies()`/`headers()`, or a page's `params`/`searchParams` outside a `<Suspense>` boundary fails with *"Uncached data was accessed outside of `<Suspense>`"* — wrap the dynamic subtree (or add `loading.tsx`), don't make the whole route dynamic.
- **Where the cache lives.** The default handler is in-memory. On a long-running server entries persist across requests; on **serverless, entries usually don't survive between requests** — `'use cache'` there mainly feeds the build-time static shell. If runtime reuse matters on serverless, `'use cache: remote'` uses a platform cache handler (a network round-trip and usually a fee). No cache entry survives a new deploy. `'use cache: private'` exists for the rare case where runtime values can't be passed as arguments — prefer refactoring.
- **Draft Mode bypasses it automatically.** With Draft Mode on, every cached scope re-executes per request and nothing is saved — no special-casing needed (`strapi-next.md`).
- **Old route-segment config is gone in this model.** `export const revalidate`, `export const dynamic = 'force-static'` and `fetchCache` don't apply — `'use cache'` + `cacheLife` replaces them. Flag them in an audit of a `cacheComponents` project.
- **Client side:** the router keeps cached content for the profile's `stale` time, with a 30-second minimum — a just-saved change still needs `updateTag`/`refresh()` to show immediately.

### Invalidation after a write

| Call | Where | Effect |
| :--- | :--- | :--- |
| `updateTag('orders')` | **Server Actions only** | Expires the tag and the next read waits for fresh data — **read-your-own-writes**. The user who saved sees the change. |
| `revalidateTag('plans', 'max')` | Server Actions, Route Handlers (webhooks) | Marks the tag stale; the next visitor gets the cached value while it refreshes in the background. Next 16 requires the second argument — a profile name (`'max'` recommended) or `{ expire: seconds }`; the one-argument form is deprecated. |
| `revalidatePath('/billing')` | Server Actions, Route Handlers | Invalidates a path's cached output. Prefer tags — they follow the data, not the URL. |
| `refresh()` (from `next/cache`) | Server Actions only | Re-renders **uncached** data on the current page (a header count, live metrics) without touching any cache. |

Rule of thumb: **a user editing their own data → `updateTag`. A CMS webhook or background job → `revalidateTag(tag, 'max')`.** Every tag you `cacheTag` must have a write that invalidates it, or it is stale data on a timer.

**Every write names how each view of its data refreshes.** For each page or component that shows the changed data, say which applies: a cached read → `updateTag` its tag; an uncached (dynamic) read → the next request is fresh, so a `redirect()` or `refresh()` is enough — say so in one line; a TanStack Query read → `invalidateQueries`. "It redirects, so it's fine" is only true until someone adds `'use cache'` to that read — tag the read when you write it, and the write stays correct.

### Older projects (Next 14/15, or `cacheComponents` off)

Follow the project's model and say which one it is: `fetch(url, { next: { revalidate, tags } })`, `unstable_cache`, route-segment `revalidate`/`dynamic`. In Next 15, `fetch` is uncached by default; in 14 it was cached by default — a migration between them changes behaviour silently. Do not sprinkle `'use cache'` into a project that hasn't enabled it.

---

## 3. Reading in Server Components

```tsx
// app/(app)/orders/page.tsx
import { Suspense } from 'react'
import { listOrdersForCurrentUser } from '@/features/orders/server/orders'

export default function OrdersPage() {
  return (
    <Suspense fallback={<OrdersTableSkeleton />}>
      <OrdersTable />
    </Suspense>
  )
}

async function OrdersTable() {
  const orders = await listOrdersForCurrentUser()     // DAL: authorises, scopes, selects (auth.md)
  if (!orders.length) return <OrdersEmpty />
  return <OrdersTableView orders={orders} />
}
```

- **Fetch where the data is used**, inside the Suspense boundary that shows its skeleton. A page that awaits everything at the top blocks the whole shell on the slowest query.
- **Start independent reads in parallel** — `Promise.all`, or start the promise early and `await` later. Sequential `await`s on unrelated data are a waterfall.
- **Pass a promise to a Client Component** and unwrap it with `use()` when the client needs the data but the server should start the request.
- **`React.cache()`** dedupes the same call within one request (e.g. `getCurrentUser()` called by the layout and the page). It is per-request memoisation, not a cross-request cache — that is `'use cache'`.
- Four states still apply: loading (Suspense fallback), empty, error (`error.tsx` for thrown, an inline message for expected), success.

---

## 4. TanStack Query — the client cache layer

Use it once the client genuinely needs a cache: data refetched on focus or interval, infinite lists, optimistic updates across components, or a client-heavy dashboard. That is the threshold — not "it looks nicer than a Server Component".

### Setup — one client per server request, one per browser

```tsx
// lib/query-client.ts — mirrors TanStack's official App Router setup
import { QueryClient, defaultShouldDehydrateQuery, isServer } from '@tanstack/react-query'
// Newest v5 releases replace `isServer` with `environmentManager.isServer()` — use whichever the installed version exports

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { staleTime: 60_000 },   // > 0, or prefetched data refetches immediately on the client
      dehydrate: { shouldDehydrateQuery: (q) => defaultShouldDehydrateQuery(q) || q.state.status === 'pending' },
    },
  })
}

let browserClient: QueryClient | undefined

export function getQueryClient() {
  if (isServer) return makeQueryClient()            // never share a cache between users' requests
  return (browserClient ??= makeQueryClient())
}
```

```tsx
// app/providers.tsx
'use client'
import { QueryClientProvider } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/query-client'

export function Providers({ children }: { children: React.ReactNode }) {
  // getQueryClient(), not useState: React discards a useState client if something suspends with no boundary in between
  const queryClient = getQueryClient()
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
}
```

### Query options defined once

```ts
// features/orders/queries.ts
import { queryOptions } from '@tanstack/react-query'
import { fetchJson } from '@/lib/api'

export const orderKeys = {
  all: ['orders'] as const,
  list: (page: number) => [...orderKeys.all, 'list', page] as const,
  detail: (id: string) => [...orderKeys.all, id] as const,
}

export const ordersListOptions = (page: number) =>
  queryOptions({
    queryKey: orderKeys.list(page),
    queryFn: () => fetchJson<OrderPage>(`/api/orders?page=${page}`),
  })
```

Define keys and options **once, in the feature**, and import them. A key literal duplicated in two components is how cache bugs are born.

### Prefetch on the server, hydrate on the client

```tsx
// app/(app)/orders/page.tsx
import { HydrationBoundary, dehydrate } from '@tanstack/react-query'
import { getQueryClient } from '@/lib/query-client'
import { ordersListOptions } from '@/features/orders/queries'

export default function OrdersPage() {
  return (
    <Suspense fallback={<OrdersListSkeleton />}>   {/* required with cacheComponents: the prefetch is request-time data */}
      <PrefetchedOrders />
    </Suspense>
  )
}

async function PrefetchedOrders() {
  const queryClient = getQueryClient()
  await queryClient.prefetchQuery(ordersListOptions(1))
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <OrdersList />   {/* 'use client' — useQuery(ordersListOptions(page)) finds the data already there */}
    </HydrationBoundary>
  )
}
```

TanStack's docs also show a streaming variant (start the query without `await`, let `pending` queries dehydrate) — use it when the shell should not wait at all.

### Two statuses, two questions

- `status` — `'pending' | 'error' | 'success'`: is there data yet?
- `fetchStatus` — `'fetching' | 'paused' | 'idle'`: is a request in flight right now?

A background refetch of rendered data is `status: 'success'` + `fetchStatus: 'fetching'` — show a subtle indicator, not a full skeleton. Getting this wrong is the most common TanStack Query mistake.

### Mutations + invalidation

```ts
const queryClient = useQueryClient()

const { mutate: createOrder, isPending } = useMutation({
  mutationFn: async (input: CreateOrderInput) => {
    const result = await createOrderAction(input)
    if (!result.ok) throw result                     // getErrorMessage reads ActionResult failures
    return result.data
  },
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: orderKeys.all })   // prefix match — every ['orders', …]
    toast.success('Order created')
  },
  onError: (error) => toast.error(getErrorMessage(error)),      // backend-owned message (backend-errors.md)
})
```

- Keys are matched by **prefix** — structure them broadest → narrowest so one invalidation sweeps a domain.
- `mutate` is fire-and-forget (errors land in `onError`); `mutateAsync` rejects — if you use it, you own the `try/catch`.
- Disable the submit button on the mutation's `isPending`, never on the query's `status`.
- **Server Actions are fine as `mutationFn`; avoid them as `queryFn`.** Actions are POST requests that the client runs one at a time — using them for reads serialises every query. Reads go through a Route Handler or a server prefetch.
- If the same data is also rendered by a Server Component, the action must invalidate the server side too (`updateTag`), or the two views disagree.

### One resource, one mechanism

A list read through `useQuery` and written through a Server Action that only calls `updateTag` will not refresh the client cache — and a list rendered by a Server Component won't notice a TanStack invalidation. Pick the owner per resource and invalidate *that* owner after every write.

---

## 5. Zustand — client UI state

```ts
// features/builder/store.ts
import { createStore } from 'zustand/vanilla'

export type BuilderState = { step: number; selectedIds: string[] }
export type BuilderActions = { next: () => void; toggle: (id: string) => void; reset: () => void }

const initial: BuilderState = { step: 0, selectedIds: [] }

export const createBuilderStore = (init: Partial<BuilderState> = {}) =>
  createStore<BuilderState & BuilderActions>()((set) => ({
    ...initial,
    ...init,
    next: () => set((s) => ({ step: s.step + 1 })),
    toggle: (id) => set((s) => ({
      selectedIds: s.selectedIds.includes(id) ? s.selectedIds.filter((x) => x !== id) : [...s.selectedIds, id],
    })),
    reset: () => set(initial),
  }))
```

```tsx
// features/builder/store-provider.tsx
'use client'
import { createContext, useContext, useState } from 'react'
import { useStore } from 'zustand'
import { createBuilderStore, type BuilderState, type BuilderActions } from './store'

type BuilderStore = ReturnType<typeof createBuilderStore>
const BuilderStoreContext = createContext<BuilderStore | null>(null)

export function BuilderStoreProvider({ init, children }: { init?: Partial<BuilderState>; children: React.ReactNode }) {
  const [store] = useState(() => createBuilderStore(init))   // one store per mounted tree, never per module
  return <BuilderStoreContext.Provider value={store}>{children}</BuilderStoreContext.Provider>
}

export function useBuilderStore<T>(selector: (s: BuilderState & BuilderActions) => T): T {
  const store = useContext(BuilderStoreContext)
  if (!store) throw new Error('useBuilderStore must be used inside <BuilderStoreProvider>')
  return useStore(store, selector)
}
```

- **Provider per tree, not a module singleton** — this is Zustand's official Next.js pattern. A Next.js server handles many requests at once, so a module-level store is shared between them — one user's state can render into another user's HTML.
- **Server Components never read or write the store.** They can't use hooks or context; pass server data into the provider's `init` from a Server Component parent instead.
- **Select narrowly** (`useBuilderStore((s) => s.step)`), never the whole store — every subscriber re-renders on every change otherwise.
- Actions live in the store; components never `setState` it from outside.
- **Not for server data** (§1). Not for URL state — `nuqs` owns filters and pagination.
- A store holding one boolean is a `useState`.
