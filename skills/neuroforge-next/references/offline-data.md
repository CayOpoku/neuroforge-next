# Offline Data, Dexie & PWA Persistence

Read before installing Dexie, writing an IndexedDB schema, or adding an offline fallback to any query. Assumes `data-fetching.md` — TanStack Query owns the network cache; this file only covers what survives a reload with no network.

---

## 1. Mode gate — resolve this before writing any data layer

**Do not infer the mode.** A `dexie` entry in `package.json` is compatible with all three modes and proves nothing.

Check `neuroforge/00-project-overview.md` for a recorded **Offline Mode**. If it is recorded, follow it and do not ask again. If it is not, ask once — one question, three options — and stop for the answer.

This is the one question this skill asks at any tier: it is an architecture decision the user owns, it is asked exactly once per project, and getting it wrong means rewriting the data layer.

| Mode | Source of truth | Reads offline | Writes offline | Default patterns |
| :--- | :--- | :--- | :--- | :--- |
| **Online-only** | Server | ✗ | ✗ | `data-fetching.md` only. **Do not install Dexie.** |
| **Hybrid** | Server | ✓ (last known) | ✗ (queued or blocked) | §4 for server data, §5 for drafts/settings |
| **Local-first** | Dexie | ✓ | ✓ (outbox → sync) | §5 everywhere; the network is a background sync job |

Record the answer in `00-project-overview.md` (append, never overwrite — `project-memory.md`):

```markdown
## Offline Mode
**Hybrid** — decided 2026-10-06. Server owns catalogue + orders; Dexie caches
reads for offline viewing. Drafts and user settings are local-first (§5).
```

**Choosing when the user asks you to pick:** most SaaS dashboards are **online-only** — TanStack Query's cache already covers a flaky connection, and Dexie is a second source of truth to keep consistent forever. Escalate to **hybrid** only when the app must render real data with the network fully down. Reserve **local-first** for apps whose primary job happens offline (field capture, note-taking, POS) — it is the only mode that forces you to own conflict resolution.

A lighter hybrid option: TanStack Query's persister (`@tanstack/react-query-persist-client` with an IndexedDB or storage persister) persists the query cache itself. It suits "show the last data offline" without a hand-designed schema; Dexie (§4–5) suits data you query, index, or write offline.

Per-table overrides are normal. Drafts and local settings are local-first in every mode above online-only.

---

## 2. Guardrails

- **Single source of truth per table.** A table is owned by the server (via TanStack Query / Server Components) or by Dexie — never both. Never mirror a Dexie row into `useState` and keep the two in sync by hand.
- **Validate at the boundary, not everywhere.** Every payload crossing *into* persistence is parsed (§6). Reads from your own tables are trusted unless a schema version changed.
- **Degrade, never fabricate.** An offline fallback returns the last known row and says so in the UI. It never returns an empty object, a zero, or a placeholder shaped like success.
- **No conflict-resolution engine.** Last-write-wins on `updatedAt`, plus an outbox, until the user asks for more.

---

## 3. Server rendering: Dexie is client-only

IndexedDB and `navigator` do not exist on the server. **A Dexie call during the server render of a Client Component crashes it** — the most common way this layer breaks in Next.js.

- Dexie is only touched from **Client Components**, inside effects, event handlers, `useLiveQuery`, or a `queryFn` that runs in the browser.
- The DB module is imported only by client files. Keep it in `lib/local-db/` and never import it from a Server Component or a `server/` module.
- Guard the helpers in one place:

```ts
// lib/local-db/cache.ts
import type { Table } from 'dexie'

const isBrowser = typeof window !== 'undefined'

export async function readLocal<T, K>(table: Table<T, K>, id: K): Promise<T | undefined> {
  if (!isBrowser) return undefined
  return table.get(id)
}

export async function writeLocal<T, K>(table: Table<T, K>, row: T): Promise<void> {
  if (!isBrowser) return
  await table.put(row)
}
```

Also: **never branch on `navigator.onLine` for correctness.** It reports whether an interface is up, not whether your API is reachable. Use it for *UI state* (a banner, a disabled button) and let a failed request be the authority on reachability.

---

## 4. Pattern A — hybrid query (network-first, durable fallback)

For server-owned data that must still render with the network down.

```ts
// features/catalogue/queries.ts
import { queryOptions } from '@tanstack/react-query'
import { z } from 'zod'
import { fetchJson } from '@/lib/api'
import { localDb } from '@/lib/local-db'
import { readLocal, writeLocal } from '@/lib/local-db/cache'

export const ProductSchema = z.object({ id: z.string(), name: z.string(), priceCents: z.number(), updatedAt: z.string() })
export type Product = z.infer<typeof ProductSchema>

export const productOptions = (id: string) =>
  queryOptions({
    queryKey: ['products', id],
    queryFn: async (): Promise<Product> => {
      try {
        const product = ProductSchema.parse(await fetchJson(`/api/products/${id}`))   // parse on write, once
        await writeLocal(localDb.products, product)
        return product
      } catch (networkError) {
        const cached = await readLocal(localDb.products, id)
        if (cached) return cached                                                  // trusted: parsed on the way in (§6)
        throw networkError                                                          // no data at all — surface it
      }
    },
    networkMode: 'offlineFirst',   // run the queryFn even when the browser reports offline, so the catch path can serve the cache
  })
```

In a Client Component: `const { data, error, status } = useQuery(productOptions(id))`.

- **The cache is read in `catch`, not up front** — no IndexedDB round-trip on the hot path.
- **No `navigator.onLine` pre-check** — the `catch` covers offline, DNS failure, 5xx and captive portals in one path.
- **`networkMode: 'offlineFirst'`** — TanStack Query otherwise pauses the query while the browser reports offline, and your fallback never runs. Verify the option on the installed version.
- Tell the user the data is stale: *"Offline — showing data from 14:02."*

---

## 5. Pattern B — local-first table + outbox

For rows the client creates and owns: drafts, settings, queued actions, and every table in local-first mode.

```ts
// lib/local-db/index.ts
import Dexie, { type Table } from 'dexie'
import type { Product } from '@/features/catalogue/queries'

export interface Task { id: string; title: string; createdAt: string; updatedAt: string }
export interface OutboxEntry {
  id?: number
  op: 'create' | 'update' | 'delete'
  table: string
  payload: unknown
  createdAt: string
  attempts: number
}

class LocalDb extends Dexie {
  tasks!: Table<Task, string>
  products!: Table<Product, string>
  outbox!: Table<OutboxEntry, number>

  constructor() {
    super('app-db')
    this.version(1).stores({
      tasks: 'id, createdAt',          // first field = primary key, the rest are indexes
      products: 'id, updatedAt',
      outbox: '++id, createdAt',
    })
  }
}

export const localDb = new LocalDb()
```

Bind a table straight to the UI with a live query — it re-renders on every write, with no manual refetch and no store mirroring:

```tsx
'use client'
import { useLiveQuery } from 'dexie-react-hooks'
import { localDb } from '@/lib/local-db'

export function TaskList() {
  const tasks = useLiveQuery(() => localDb.tasks.orderBy('createdAt').reverse().toArray(), [])
  if (tasks === undefined) return <TaskListSkeleton />     // undefined = still loading, not empty
  if (!tasks.length) return <TaskListEmpty />
  return <ul>{tasks.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
}
```

**Writes go to the table and the outbox in one transaction**, so a crash cannot leave a local row that will never sync:

```ts
export async function createTask(input: Pick<Task, 'title'>) {
  const now = new Date().toISOString()
  const task: Task = { id: crypto.randomUUID(), title: input.title, createdAt: now, updatedAt: now }

  await localDb.transaction('rw', localDb.tasks, localDb.outbox, async () => {
    await localDb.tasks.add(task)
    await localDb.outbox.add({ op: 'create', table: 'tasks', payload: task, createdAt: now, attempts: 0 })
  })
}
```

Flush the outbox when connectivity returns: oldest first, through a Server Action or Route Handler that validates and authorises like any other write; delete each entry only after the server confirms it; cap `attempts` so one poison entry cannot loop forever. Server wins on `updatedAt` conflicts unless the user specifies otherwise.

---

## 6. Zod placement — parse on write, `safeParse` on version change

| Boundary | Rule |
| :--- | :--- |
| Network → app | `.parse()` — the payload is untrusted. Failure is a real error. |
| App → Dexie | Already parsed above; do not parse twice. |
| Dexie → app, same schema version | Trust it. No parse. |
| Dexie → app, after a schema version bump | `.safeParse()` per row, **drop** invalid rows, never throw the whole query. |

```ts
const rows = await localDb.products.toArray()
const valid = rows.flatMap((row) => {
  const parsed = ProductSchema.safeParse(row)
  return parsed.success ? [parsed.data] : []
})
```

Bump `this.version(n)` whenever a stored shape changes, and add the Dexie upgrade function in the same commit as the Zod schema change. A schema edit without a version bump ships silent data corruption to everyone who already holds the old DB.

**PWA:** a service worker for offline page loads is a separate concern (`@serwist/next` or the project's choice). It caches the shell and assets; it does not replace this file's data layer, and stale service workers cause their own "works in Incognito" bugs — version them and offer an update prompt.

---

## 7. Smells

| Smell | Why it is wrong |
| :--- | :--- |
| Dexie imported by a Server Component, `server/` module, or touched during render | Crashes the server render |
| `if (navigator.onLine)` deciding whether to fetch | Reports the interface, not your API |
| Same data read via `useQuery` and written via raw `localDb.table.put` | Two sources of truth; the cache goes stale silently |
| `Schema.parse()` on every Dexie read | Re-validating data you wrote and already parsed |
| Dexie schema edited without a `version()` bump | Silent corruption for existing users |
| Server data overwriting a local row with no `updatedAt` comparison | Destroys an unsynced local edit |
| Offline fallback returning `{}` or `0` | Fabricated success — the UI cannot tell it from real data |
| `useLiveQuery` result `undefined` treated as empty | Loading shown as "no data" |
| Dexie installed in an online-only app | A second source of truth to maintain forever, for no gain |
