# Rendering: Derive First, Effects Last

The default answer to "how do I make this update" is **compute it during render**. `useEffect` is an escape hatch for synchronising with something *outside* React — a browser API, a third-party widget, a subscription. It is not a tool for deriving values, and it is not a data-fetching tool.

An effect that sets state from other state creates a second source of truth, renders one frame late, and silently stops matching its input the first time someone adds a third way to change it. A derived value cannot go stale.

---

## 1. The hierarchy

1. **Derive during render** — a `const` computed from props and state. Almost always the answer.
2. **Move it to the server** — if it needs data, a Server Component reads it before the client ever renders.
3. **The URL** — filters, tabs, pagination, search: state that should survive a reload and a shared link lives in search params (`nuqs` or `useSearchParams`).
4. **An established hook** — a browser API, timer, listener or observer someone has already wrapped correctly (`useSyncExternalStore`, the project's hooks library).
5. **`useEffect`** — a genuine side effect that rendering cannot express.

Do not skip levels. If you are at 5, be able to say in one sentence why 1–4 did not work.

---

## 2. What each is actually for

| You want to | Use |
| :--- | :--- |
| Derive a value from state or props | A `const` in the render body |
| Derive an expensive value | `useMemo` — **only when measured**, or let the React Compiler handle it if enabled |
| Format, filter, sort, sum, group | A `const` (or `useMemo` past a measured cost) |
| Reset state when a prop changes | A `key` on the component — never an effect copying the prop |
| Load data the page needs | A Server Component (`data-fetching.md`) |
| Refetch on the client when an input changes | Put the input in the **query key** (`useQuery({ queryKey: ['orders', page] })`) — never an effect that calls `refetch()` |
| Respond to a user action | The event handler — not an effect watching state the handler set |
| Filters / tabs / page number | Search params (`nuqs`), so the URL is the state |
| Read a browser API (`matchMedia`, online status, window size) | `useSyncExternalStore`, or the project's hooks library |
| Persist a value across reloads that the server must render | A cookie read on the server — `localStorage` cannot render on the server |
| Fire analytics on a state transition | The event handler that caused it; an effect only if the transition has no single handler |
| Push to a non-React instance (map, chart, editor) | `useEffect` with cleanup |
| Initialise a browser-only library | `useEffect`, or `next/dynamic` with `ssr: false` from a Client Component |

---

## 3. Anti-patterns

```tsx
// Wrong — an effect maintaining derived state
const [fullName, setFullName] = useState('')
useEffect(() => { setFullName(`${first} ${last}`) }, [first, last])

// Right
const fullName = `${first} ${last}`
```

```tsx
// Wrong — fetching in an effect: no SSR, a waterfall, a flash of empty state, race conditions
const [orders, setOrders] = useState<Order[]>([])
useEffect(() => { fetch('/api/orders').then((r) => r.json()).then(setOrders) }, [])

// Right — read it on the server (data-fetching.md)
export default async function OrdersPage() {
  const orders = await listOrdersForCurrentUser()
  return <OrdersTable orders={orders} />
}
```

```tsx
// Wrong — copying a prop into state and syncing it (desyncs the moment the parent updates)
const [value, setValue] = useState(props.value)
useEffect(() => setValue(props.value), [props.value])

// Right — controlled: use the prop. Or, to reset local edits when the record changes:
<EditForm key={record.id} record={record} />
```

```tsx
// Wrong — an effect that refetches when an input changes
useEffect(() => { refetch() }, [page])

// Right — the key carries the dependency, so the refetch is automatic
const { data } = useQuery({ queryKey: ['orders', page], queryFn: () => listOrders(page) })
```

```tsx
// Wrong — an effect reacting to state the handler just set
const [submitted, setSubmitted] = useState(false)
useEffect(() => { if (submitted) toast.success('Saved') }, [submitted])

// Right — do it in the handler that knows it happened
async function onSubmit() {
  const result = await saveAction(data)
  if (result.ok) toast.success('Saved')
}
```

```tsx
// Wrong — manual listener plumbing for a browser value
const [isWide, setIsWide] = useState(false)
useEffect(() => {
  const mq = window.matchMedia('(min-width: 1024px)')
  const on = () => setIsWide(mq.matches)
  on(); mq.addEventListener('change', on)
  return () => mq.removeEventListener('change', on)
}, [])

// Right — subscribe properly, with a server snapshot
function useMediaQuery(query: string) {
  return useSyncExternalStore(
    (cb) => { const mq = window.matchMedia(query); mq.addEventListener('change', cb); return () => mq.removeEventListener('change', cb) },
    () => window.matchMedia(query).matches,
    () => false,   // server snapshot — and prefer CSS breakpoints for layout anyway
  )
}
```

---

## 4. Memoisation

- **Don't memoise by reflex.** `useMemo` / `useCallback` / `memo` cost code and only pay off for a measured slow render or a referentially-sensitive dependency.
- **React Compiler:** if `reactCompiler: true` is set in `next.config`, the compiler memoises for you. Do not add manual `useMemo`/`useCallback` to compiled components without a reason; check the config before recommending either way.
- A value used as an effect dependency or passed to a memoised child is the legitimate case — say so in one line when you add it.

---

## 5. Hooks libraries

Use what the project already depends on (`usehooks-ts`, `@uidotdev/usehooks`, `react-use`, ahooks). Do not add a hooks library for one hook — a 10-line `useSyncExternalStore` wrapper is cheaper. When recommending one, suggest the install command and let the developer run it.

**SSR caveat:** anything reading `window`, `document`, `matchMedia` or `localStorage` has no server value. A hook returning a different first value on the client than the server rendered is a hydration mismatch. Prefer CSS for responsive layout; for values that must be right in the first HTML, read a cookie on the server.

---

## 6. When `useEffect` is the right answer

Legitimate uses — all synchronisation with something outside React:

- Syncing to a non-React instance (map, chart, rich-text editor) — with cleanup.
- Subscribing to an external source that has no hook (prefer `useSyncExternalStore`).
- Imperative DOM work after render: focus, scroll a list to a new selection, measure.
- Persisting a draft, debounced.

Rules when you do use one:
- **Always return a cleanup** for listeners, timers, subscriptions and in-flight requests (`AbortController`).
- **Narrow dependencies.** Depend on `user.email`, not the whole `user` object recreated every render.
- **Never chain effects.** An effect that sets state that triggers another effect is a loop waiting for the right input — collapse it into the handler or a derived value.
- **Strict Mode runs effects twice in development.** If that breaks something, the effect is missing a cleanup — fix it, don't disable Strict Mode.
- `useEffectEvent` (React 19.2) separates "read the latest value" from "re-run when it changes" — use it instead of omitting dependencies, if the installed React has it.
