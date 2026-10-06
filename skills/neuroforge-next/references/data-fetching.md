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

- **`'use cache'`** goes at the top of an async function, a component, or a file. The cache key is built from the arguments and closed-over values — keep them **small and serialisable**, and never pass a whole request object.
- **No request data inside a cached scope.** `cookies()`, `headers()` and `searchParams` cannot be read inside `'use cache'` — read them outside and pass the specific value in as an argument (and accept that each distinct value is its own cache entry). Per-user data is usually not a `'use cache'` candidate at all.
- **The static shell.** With `cacheComponents`, `'use cache'` output plus `<Suspense>` fallbacks form the prerendered shell (Partial Prerendering); everything else streams. Request-time data accessed outside a `<Suspense>` boundary is a build error — wrap the dynamic subtree, don't make the whole route dynamic.
- **Old route-segment config is gone in this model.** `export const revalidate`, `export const dynamic = 'force-static'` and `fetchCache` don't apply — `'use cache'` + `cacheLife` replaces them. Flag them in an audit of a `cacheComponents` project.
- Variants such as `'use cache: private'` / `'use cache: remote'` exist in newer releases — **verify against the installed version's docs** before recommending one.

### Invalidation after a write

| Call | Where | Effect |
| :--- | :--- | :--- |
| `updateTag('orders')` | **Server Actions only** | Expires the tag and the next read waits for fresh data — **read-your-own-writes**. The user who saved sees the change. |
| `revalidateTag('plans', 'max')` | Server Actions, Route Handlers (webhooks) | Marks the tag stale; the next visitor gets the cached value while it refreshes in the background. Next 16 expects the second `cacheLife` profile argument. |
| `revalidatePath('/billing')` | Server Actions, Route Handlers | Invalidates a path's cached output. Prefer tags — they follow the data, not the URL. |
| `refresh()` | Server Actions | Refreshes the client router's view of the current page without touching tags. |

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
// lib/query-client.ts
import { QueryClient, defaultShouldDehydrateQuery, isServer } from '@tanstack/react-query'

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

export default async function OrdersPage() {
  const queryClient = getQueryClient()
  await queryClient.prefetchQuery(ordersListOptions(1))
  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <OrdersList />   {/* 'use client' — useQuery(ordersListOptions(page)) finds the data already there */}
    </HydrationBoundary>
  )
}
```

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

- **Provider per tree, not a module singleton,** whenever the store is initialised from server data or rendered during SSR. A module-level store is shared by every request on the server — one user's state can render into another user's HTML.
- **Select narrowly** (`useBuilderStore((s) => s.step)`), never the whole store — every subscriber re-renders on every change otherwise.
- Actions live in the store; components never `setState` it from outside.
- **Not for server data** (§1). Not for URL state — `nuqs` owns filters and pagination.
- A store holding one boolean is a `useState`.
