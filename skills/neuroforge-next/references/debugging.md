# Server/Client Contexts, Diagnostics, Logging & Debugging Protocol

Use this reference to diagnose runtime issues, design logs, and fix hydration errors.

---

## 1. Console Log Debugging Workflow

When investigating bugs, unexpected behaviour, or data flow issues:
1. **Lean on `console.log` diagnostics.** Instead of burning context guessing or making speculative changes, insert targeted logs to observe actual runtime values and execution paths.
2. **Clean up.** Once the root cause is found and fixed, remove the temporary logs.

### Why frontend bugs cost more — and the fix

A server bug has ground truth you can reach alone: throw, log, typecheck, read the terminal. **A client bug's ground truth is in the developer's browser, on their screen.** You cannot see it. The failure mode is substituting the only thing you *can* do alone — reading more files, guessing wider — for the one observation that would settle it.

So make the developer the instrument. They are sitting in front of the running app:

1. **Write the log, don't hunt for the answer.** One or two labelled `console.log`s at the exact point where the value should be right.
2. **Tell them precisely where it will print.** Server Component, Server Action and Route Handler logs print in the **terminal running `next dev`**; Client Component logs print in the **browser console**. *"Load `/dashboard`, click Save, paste what `DEBUG_SAVE:` prints in the terminal."*
3. **Stop and wait.** Do not read more files while waiting. Do not ship a speculative fix "in the meantime".
4. **Read the output, then decide.**

**Before a second fix attempt, you must have an observation** — a log, an error, something they saw. Two fixes for one symptom with no new evidence between them means stop guessing and instrument (`SKILL.md` → Diagnose mode).

Ask for what only they can see: the exact error text, what the Network tab shows, whether it happens in `next dev` or only after `next build && next start`, whether it ever worked, what changed since.

---

## 2. Explicit confirmation for diagnostics, lint and build commands

- **Lint:** the project's `lint` script (`eslint .`). `next lint` was removed in Next 16 — if the script still calls it, say so and suggest `eslint .`.
- **Typecheck:** `npx tsc --noEmit` (or the project's `typecheck` script).
- **Build:** `next build` catches route-level errors (dynamic data outside Suspense under `cacheComponents`, invalid exports, parallel-route slots missing `default.tsx`) that `tsc` cannot — but it is slow and writes `.next/`. Ask first. `next build --debug-prerender` gives stack traces naming the failing component.
- **Ask before running any of them.** Say which command and why.
- **Zero `any`:** never as an escape hatch. `unknown` and narrow — `type-safety.md`.

### Reading a hydration error

React 19 reports a diff of the mismatched text or attribute. Work backwards:
1. Note the element and the server vs client values in the error.
2. Find what feeds it — a prop, a derived value, a store, a hook.
3. Ask what could differ between server and client: time, randomness, `window`, `localStorage`, a locale/timezone-dependent format, a browser extension injecting attributes, invalid HTML nesting.
4. Fix the source with the table in §5. Never silence it with `suppressHydrationWarning` (except the one documented theme case) or by wrapping everything in `dynamic(..., { ssr: false })`.

---

## 3. Server / client context rules

Know which side a line runs on before fixing a runtime error:

| File / code | Runs on |
| :--- | :--- |
| Server Component (no directive), `page.tsx`/`layout.tsx` by default | Server only |
| Server Action (`'use server'`), Route Handler, `proxy.ts` | Server only |
| Client Component (`'use client'`) | **Server for the first HTML render, then the browser** |
| `useEffect` body, event handlers | Browser only |

The third row is the trap: a Client Component still renders on the server, so `window` at the top of its body crashes SSR. `typeof window === 'undefined'` checks inside render cause hydration mismatches — move browser reads into an effect, an event handler, `useSyncExternalStore`, or `dynamic(..., { ssr: false })`.

Prefix logs so they can be filtered:

```ts
const side = typeof window === 'undefined' ? '[SERVER]' : '[CLIENT]'
console.log(`${side} DEBUG_USER:`, user)
```

---

## 4. Structured logging rules (agent-optimised)

```ts
// ❌ Bare output, hard to scan
console.log(data)

// ✅ Instant grepping and identification
console.log('DEBUG_DATA_FETCH:', { payload: data, at: Date.now() })

// ✅ Tabular rows
console.table(items)

// ✅ Deep server objects
console.log(JSON.stringify(serverData, null, 2))

// ✅ Performance tracing
console.time('orders-query')
const orders = await db.order.findMany(/* … */)
console.timeEnd('orders-query')
```

Never log secrets, tokens, or full request bodies containing PII. In production, log the error digest alongside the context so it can be matched to what the user saw.

---

## 5. Hydration mismatch protocol

Never ignore a hydration error. It discards the server HTML for that subtree, re-renders on the client, and can break interactivity.

| Problem | Wrong | Right |
| :--- | :--- | :--- |
| **Browser-only API** | `localStorage.getItem('theme')` in render | Cookie read on the server, or `next-themes` |
| **Randomness** | `Math.random()` / `crypto.randomUUID()` in render | Generate on the server and pass it down; `useId()` for element ids |
| **Time** | `new Date().toLocaleString()` in render | Format on the server with explicit `locale` + `timeZone`, or render after mount in a small client component |
| **Client-only condition** | `typeof window !== 'undefined' && window.innerWidth > 768` in render | CSS media queries; `useSyncExternalStore` with a server snapshot |
| **Browser-only library** | Imported at module scope | `dynamic(() => import(...), { ssr: false })` from a Client Component |
| **Invalid HTML nesting** | `<div>` inside `<p>`, `<a>` inside `<a>` | Fix the markup |
| **Theme class on `<html>`** | — | `suppressHydrationWarning` on `<html>` only, with `next-themes` — the one sanctioned use |

---

## 6. Error handling & boundaries

- **Expected errors are returned, unexpected errors are thrown** (`backend-errors.md` §2).
- **`error.tsx`** (a Client Component) catches thrown errors in its segment; `global-error.tsx` catches errors in the root layout. In production, server error messages are redacted — `error.digest` matches the server log line.
- **`notFound()`** renders the nearest `not-found.tsx` with a 404. Use it for missing resources.
- **`redirect()` and `notFound()` throw by design** — a `try/catch` around them swallows the navigation. Call them outside the `try`, or rethrow.
- Client Component errors in event handlers are not caught by `error.tsx` (boundaries only catch render errors) — handle them where they happen and surface the message.

---

## 7. Works locally, broken in production

`next dev` compiles on demand, reads `.env` live, and never caches across deploys. A production-only failure is usually the build, the host environment, the cache model, or the visitor's stale client. Read the signature before reading code.

| Signature | Meaning | Cheapest separator |
| :--- | :--- | :--- |
| A `NEXT_PUBLIC_*` change has no effect | Public env is **inlined at build time** | Rebuild and redeploy — a restart is not enough |
| A server env change has no effect | Process not restarted, or the value was read during prerender / inside `'use cache'` | Restart; if still stale, check whether the page is static |
| Data is stale after a save, fine in dev | The page is prerendered or cached, and the write doesn't invalidate it | Does the action call `updateTag`/`revalidatePath` for what the page reads? |
| `Error: Failed to find Server Action "…"` | **Version skew**: a browser loaded before the deploy is calling an action id the new build doesn't have; or instances built separately | Hard refresh fixes it for that user → skew. Fix: deploy-level skew protection (Vercel) or a consistent `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` + `deploymentId` across instances |
| `ChunkLoadError` / "Loading chunk … failed" after a deploy | Old client requesting chunks the new deploy removed | Same page in Incognito works → stale client. Keep previous build assets available briefly, or reload on chunk error |
| "Uncached data was accessed outside of `<Suspense>`" | `cacheComponents` on; an uncached fetch/DB read, `cookies()`/`headers()`, or a page awaiting `params`/`searchParams` sits outside a Suspense boundary | Wrap that subtree in `<Suspense>`, add `loading.tsx`, cache the read, or add `generateStaticParams` (`layouts-routing.md` §3). Only in CI? `next build --debug-prerender` names the component |
| Build hangs ~50 s then *"Filling a cache during prerender timed out"* | A request-time Promise (cookies, params, uncached data) passed into or captured by a `'use cache'` function | Await the value outside the cached scope and pass the plain value in |
| Works in build, fails at runtime with a `next-request-in-use-cache` error | `cookies()`/`headers()` called inside `'use cache'` — directly or in a helper it calls | Read it outside and pass the value as an argument (`data-fetching.md` §2) |
| An error page with only "An error occurred in the Server Components render" | An error thrown on the server; message redacted in production | Find the `digest` in the server logs |
| A form submit does a full page navigation | `<form action={serverAction}>` before hydration — this is progressive enhancement working, not a bug | If something else breaks, check the console for the first error that stopped hydration |
| New code has no effect | The deploy didn't run, or didn't go green | CI/deploy status for the merge commit before any code reading |

**Restart ≠ rebuild.** A restart relaunches the same build with a fresh server environment. A rebuild produces new client bundles (and new inlined `NEXT_PUBLIC_` values). Say which one applies in one line; developers routinely suspect the wrong one.

For the developer's own browser right now: DevTools → Application → **Clear site data**.
