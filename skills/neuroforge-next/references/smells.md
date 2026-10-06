# Codebase Smells, Audits & Quality Guards

---

## 1. Dormant files

Codebases accumulate orphaned components, unused hooks, dead utilities, unreferenced routes and stale config. During a Tier 2 analysis (`01-project-analysis.md` / `05-potential-spaghetti-risks.md`), audit for them and list them with a recommendation. `knip` finds unused files and exports quickly — suggest it rather than reading every import by hand.

Deleting dead application code during an authorised refactor is expected — that is the Boy Scout rule. The never-delete rule protects `neuroforge/` memory files, not source. Say what you removed and why.

---

## 2. No legacy patches in development

With zero live users, a temporary frontend workaround for a flawed backend contract is pure debt with no upside. Do not write one.

Recommend the clean fix. If the correct change is a backend change, say so plainly and specifically — which action or endpoint, which field, what shape it should return — so the user can act on it rather than absorbing the defect into the frontend.

---

## 3. Environment variables

### `NEXT_PUBLIC_` is a publishing decision

- **`NEXT_PUBLIC_*` values are inlined into the client bundle at build time.** Anyone can read them in DevTools. A secret with that prefix is published — a critical finding, and the key must be rotated, not just renamed.
- **They are frozen at `next build`.** Changing a `NEXT_PUBLIC_` value in the host's environment does nothing until the app is **rebuilt** — a restart is not enough (`debugging.md` §7). The same build can't serve two environments with different public values.
- **Server-only variables** (no prefix) are read at runtime by dynamic server code — set them in the host, restart. But a value read while **prerendering** a static page or a `'use cache'` scope is captured at build time too. If a server env value must change without a rebuild, read it in dynamic code.

### No silent fallbacks for mandatory values

```ts
// ✗ a misconfigured production deploy silently points at localhost
const apiUrl = process.env.API_URL || 'http://localhost:3000'
```

Fail loudly at startup for anything mandatory — `requiredEnv` (`patterns.md` §2), or one zod-validated env module imported by the server:

```ts
// server/env.ts
import 'server-only'
import { z } from 'zod'

const EnvSchema = z.object({
  DATABASE_URL: z.string().url(),
  AUTH_SECRET: z.string().min(32),
  SMTP_HOST: z.string().min(1),
})

export const env = EnvSchema.parse(process.env)   // throws on boot with every missing key listed
```

Optional values with a sensible default are fine — the rule targets values whose absence breaks the app.

`.env.example` lists every key with a comment on whether it is public, build-time or runtime. Hosting panels (cPanel, Plesk, Render…): type values without quotes — the panel keeps them as part of the value.

---

## 4. Smells to flag in an audit

1. **Dormant files** — unused components, hooks, utilities, routes.
2. **Secrets in `NEXT_PUBLIC_*`**, hardcoded keys, or `|| 'default'` fallbacks masking missing config (§3).
3. **Silenced errors** — the highest-priority finding in this list. `getErrorMessage(error) || 'Something went wrong'`, a hardcoded string in a `catch`, an empty `catch {}`, `.catch(() => null)`, a `console.error` with no UI, or a returned `{ ok: false }` that discards the upstream message (`backend-errors.md` §5).
4. **Expected errors thrown from Server Actions** — the message is redacted in production (`backend-errors.md` §2).
5. **Auth only in `proxy.ts`/`middleware.ts` or a layout** — a Server Action, Route Handler or data function that doesn't check the session itself (`auth.md`).
6. **Client-trusted ids** — a query scoped by an id from the client instead of the session; a tenant-owned query without the tenant in the `where`.
7. **Unvalidated boundaries** — a Server Action or Route Handler using its input without a schema parse; `as` on `request.json()`.
8. **`'use client'` over-reach** — on a `page.tsx` or `layout.tsx`, or high in the tree to serve one interactive leaf.
9. **Server code reachable from the client** — a DB/secret module without `import 'server-only'`, or a barrel mixing server and client exports (`structure.md` §4).
10. **Over-sharing props** — a full DB row passed into a Client Component when it renders two fields.
11. **Fetching in `useEffect`** — or an effect that derives state, copies a prop into state, or calls `refetch()` (`rendering.md` §3).
12. **Route Handler for an internal mutation** the app itself performs — that's a Server Action.
13. **Server Action used as a `queryFn`** — serialises every read (`data-fetching.md` §4).
14. **Server data in a Zustand store**, or a module-level store initialised from server data (`data-fetching.md` §5).
15. **Duplicated query keys** — the same key literal in two components instead of shared `queryOptions`.
16. **A cached read with no invalidation** — a `cacheTag` no write ever `updateTag`s or `revalidateTag`s; or legacy `export const revalidate`/`dynamic` left in a `cacheComponents` project.
17. **God components** — over ~200 lines, or fetching + interactive rendering in one file.
18. **Unbounded queries** — `findMany` with no `take`, a table with no pagination, no max on `limit`.
19. **`any` escapes** — including implicit ones from an untyped `catch` or `JSON.parse`.
20. **Hand-rolled or edited shadcn primitives** in `components/ui/` (`components.md` §5); duplicated primitive blocks instead of a `components/app/` wrapper.
21. **Comment noise** — header blocks, banners, comments restating the code, commented-out code, or any comment pointing at `neuroforge/` or the session (`code-comments.md`).
22. **Analytics misconfiguration** — the GA tag loading in development, a proxied collection path that geolocates to the server, or the ID baked with a `|| ''` fallback (`analytics.md`).
23. **Raw `<img>` for content imagery**, missing dimensions, or the LCP image lazy-loaded (`performance-a11y.md`).
24. **Soft 404s** — a "not found" component rendered with a 200 status instead of `notFound()`.
