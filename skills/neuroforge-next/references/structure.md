# Code Structure: Features, Colocation & Packages

Paths are written without `src/`; prefix them if the project uses a `src/` directory.

This file answers one question: **where does this file go, and when does the folder shape change?** Read it before adding a `features/` folder, before splitting into packages, and before proposing any restructure.

---

## 1. Small is flat. Defend it.

A small app with `app/`, `components/`, `hooks/`, `lib/` and `server/` is the correct shape until it demonstrably hurts. Having read this file is not evidence that it hurts.

**Never propose a restructure the user did not ask for.** A folder reorganisation is a diff across dozens of files that changes no behaviour, invalidates every open branch, and buys nothing a rename could not. Raise it only when the user reports the pain, or when a Tier 2 audit finds a folder past the thresholds in §4.

Packages are never the answer to "too many files". They answer "a second app appeared".

---

## 2. What goes where

| Kind | Destination | Notes |
| :--- | :--- | :--- |
| Routes, layouts, route-only UI | `app/` | `app/` is for routing. Keep it thin (`layouts-routing.md`). |
| Route-private components | `app/<segment>/_components/` | `_` prefix opts a folder out of routing. Fine for a component used by one route only. |
| Shared UI | `components/app/`, `components/<domain>/`, `components/ui/` (shadcn, CLI-only) | `components.md` |
| Client hooks | `hooks/` or `features/<x>/hooks/` | Only functions that call hooks get the `use` prefix |
| Pure functions (no React, no server) | `lib/` | Formatters, `cn()`, schemas, `fetchJson`, `getErrorMessage` — importable from both sides |
| Server-only code (DB, secrets, session, DAL) | `server/` or `features/<x>/server/` | Every file starts with `import 'server-only'` |
| Server Actions | `features/<x>/actions/` | One verb per file is fine; `'use server'` at the top |
| Types | `types/` or `features/<x>/types/` | `type-safety.md` §2 |
| Email / PDF templates | `templates/` | `email-pdf.md` |
| Scripts | `scripts/` | Never the repo root |

**The client/server split is a folder rule, not just a directive.** A file under `server/` must never be imported by a `'use client'` file; `import 'server-only'` turns that mistake into a build error instead of a leaked secret.

---

## 3. Is it a hook at all?

A bloated `hooks/` folder is usually a third functions that were never hooks.

| Test | Destination |
| :--- | :--- |
| Calls `useState`, `useEffect`, `useQuery`, `useRouter`, other hooks | `hooks/` (client) |
| Pure function — same input, same output, no React | `lib/` |
| Touches the DB, env secrets, the session | `server/` |
| Mutates data on behalf of the user | a Server Action in `features/<x>/actions/` |

The `use` prefix is reserved for hooks — `requiredEnv`, not `useRequiredEnv` (React's lint rules treat `use*` as hooks and will enforce hook rules on it).

---

## 4. `features/` — when the app has domains

**Trigger:** distinct functional domains (billing, inventory, auth) whose files are spread across `components/`, `hooks/`, `server/` and `lib/`, so one change touches five folders. Roughly: `components/` or `server/` past ~20 files with domain prefixes all over them.

Cheaper steps first:

1. **Move out what was misfiled** (§3) — pure functions out of `hooks/`, server code out of `lib/`.
2. **Rename with a domain prefix** — `billing-invoice-table.tsx`, `useBillingInvoices.ts`. Alphabetical sorting does the grouping for free.
3. **Only then, `features/`:**

```
features/
  billing/
    components/        -> invoice-table.tsx (server + client components for this domain)
    hooks/             -> use-invoice-filters.ts
    actions/           -> create-invoice.ts ('use server')
    server/            -> invoices.ts (DAL: import 'server-only')
    schemas/           -> invoice.schema.ts (zod, shared by form + action)
    types/             -> invoice.types.ts
    queries.ts         -> TanStack Query keys + options, if used
```

### Feature boundaries

- **A feature never imports another feature's internals.** `features/billing` may import `lib/`, `components/`, `server/` (shared), and another feature's **public surface** only.
- **Public surface without a barrel trap.** A single `index.ts` re-exporting everything mixes client and server modules: importing a client component through it can drag `server-only` code into the client graph (build error) or ship more JS than needed. Use **two** entry points when a feature is consumed elsewhere:
  - `features/billing/index.ts` — client-safe exports (components, hooks, types, schemas)
  - `features/billing/server.ts` — server-only exports (DAL functions, actions), starting with `import 'server-only'`
- Or skip barrels and import by path — explicit and tree-shake-friendly. Large `export *` barrels also slow dev compilation.
- Enforce with `eslint-plugin-boundaries` (or `no-restricted-imports`) once there are more than three features. A boundary nobody checks is already broken.

---

## 5. Monorepo packages — when, and when to object

**Default when splitting: by deployable app, then shared packages** — `apps/web`, `apps/admin`, `packages/ui`, `packages/db`, `packages/config`.

That axis works because the boundaries are **enforceable**: each app deploys separately and imports packages through `package.json`. A `packages/billing` that only `apps/web` uses has no such edge and degenerates into a folder with extra config.

**Object, naming which case applies, when:**

1. **One app, one audience** — a monorepo adds a build orchestrator, workspace config and versioning for nothing. Use §4.
2. **A second app doesn't exist yet** — packages are reversible; add them when the admin app or the marketing site actually splits off.
3. **The proposed package is a feature, not a shared foundation** (`billing`, `notifications`) — that's `features/` inside the app that owns it.
4. **Two apps need the same feature** — it moves down into a shared package (`packages/billing` with its own server/client entry points), never copy-pasted into both apps.
5. **It's a design system for other repos** — a publishable package is a different axis and legitimate.

Shared packages consumed by a Next app as TypeScript source need `transpilePackages` in that app's `next.config` (or a build step in the package). Verify which the workspace uses before adding one.

### Packages never import from apps, or sideways between apps

| Tier | May import from |
| :--- | :--- |
| `apps/*` | `packages/*` |
| `packages/ui`, `packages/db` | `packages/config` and external deps only |
| `packages/config` | Nothing internal |

An app importing another app's file (`../../admin/…`) is an architectural finding — move the shared code into a package.

---

## 6. Restructures are Tier 2

Moving domains into `features/`, or splitting packages, touches far more than four files: announce, scan, write the memory files, wait for "Proceed". A rename of two or three misfiled files is Tier 1.

Report the move as a mechanical diff — files moved, imports rewritten, config changed, behaviour unchanged — and suggest a codemod or script in `scripts/` for anything over ~20 files (`project-memory.md` §3).
