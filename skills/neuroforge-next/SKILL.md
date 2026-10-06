---
name: neuroforge-next
description: |
  NeuroForge Next: analysis-first engineering for premium SaaS apps on Next.js 16 (App Router first, Pages Router
  aware), React 19, Prisma and TypeScript. Severity-tiered workflow with NeuroForge memory files; enforces correct
  Server/Client boundaries, explicit Cache Components caching, authorization in the data layer, schema-validated
  Server Actions, TanStack Query + Zustand, Laws of UX, shadcn/ui wrappers, loud errors with no silent fallbacks and
  why-only comments. Use for any Next.js/React/Prisma work: components, hooks, Server Actions, Route Handlers, proxy,
  Prisma schemas, audits. Trigger on Next.js, App Router, 'use client', 'use server', 'use cache', cacheTag,
  revalidateTag, updateTag, draftMode, proxy.ts, middleware.ts, next/image, useActionState, TanStack Query, Zustand,
  nuqs, PrismaClient, zod, shadcn, hydration error, Strapi, Dexie, offline-first, Nodemailer, React Email, PDF, GA4,
  NEXT_PUBLIC_ env vars, 'Failed to find Server Action' or ChunkLoadError.
license: MIT
metadata:
  author: cayopoku
  version: "2.0.0"
---

# NeuroForge Next Protocol

Cognitive architecture for Next.js 16 / React 19 / Prisma / TypeScript. Prioritise readability, single responsibility, end-to-end type safety, correct server/client boundaries, and long-term architectural health over shortcuts.

This file is a **router**. It holds only what applies to every task. Everything else lives in `references/` and is loaded on demand — do not read a reference the current task does not need.

## Hard stops

Seven things never worth an exception. If one is in your way, say so in a line and wait.

1. **Never leave the project root.** No reading, listing, globbing or searching above it — not `C:\Users`, not the home directory, not a sibling repo. The repo is the world.
2. **Never start, restart or kill a dev server, and never open a browser.** The developer already has the app running. Ask which port.
3. **Never hand-write a shadcn/ui primitive.** `components/ui/` is CLI output only — `npx shadcn@latest add <component>`. Missing one? Surface the exact command and wait. A failing CLI is a bug to debug together, never a licence to hand-code.
4. **Never touch `.env*`, `.git/*`, `prisma/migrations/*`, lockfiles or auth/secret config** without explicit approval.
5. **Never write implementation code in a Tier 2 analysis turn.** Not one line, not "while I was in there".
6. **Never delete, overwrite or archive a file in `neuroforge/`.** Propose; the developer disposes.
7. **Never `||` a fallback onto an error utility.** `getErrorMessage(error) || 'Something went wrong'` — or any default placed after a helper whose job is to read the backend — silently replaces the real failure with a reassuring string, and the broken backend goes unreported. The utility is terminal. Fix the utility, never the call site (`references/backend-errors.md` §4).

## Working with the developer

The developer has years of Next.js experience and the app open in front of them. They can answer in five seconds what would cost you twenty file reads. **Asking is the cheap path, not the lazy one.** They know this codebase better than you do — work with them, not around them.

### Ask when the answer is cheaper than the search

Ask about what only they can see: the port, the exact error text, what is on screen, whether it ever worked, which of two files is the live one, what they already tried, whether the bug happens in `next dev` or only after `next build`.

Do not ask them to explain their own architecture, where a file lives, or what a function does. That is in the repo — read it.

- **One question per turn**, answerable in one line.
- **Only when the answer changes what you do next.** Otherwise pick the sensible default and name it in half a line.
- **No preamble.** Ask it, then stop. Do not keep working while you wait.

### Log the answer — `neuroforge/00-answers.md`

A settled question should cost once, not once per session. Read this file before asking anything; if the answer is there, use it and say nothing.

- **Ask in chat, log the answer.** Never the reverse. This file is a record of what is settled, **never a queue of open questions** — anything you need answered now goes to the developer now.
- **One line per entry, dated.** If it needs a paragraph it is analysis and belongs in a numbered file.
- **Append-only and permanent.** Never pruned, never archived. Correct a line that has gone stale by replacing that line, dated.
- Create it on first use with three headings: `## Environment` (ports, what is already running, how services start, host), `## Decisions` (settled calls not to re-propose), `## Preferences` (how the developer wants to work).

```markdown
## Environment
- Dev server runs on :3001 (`next dev --turbopack`), always already up — never start it. (2026-10-06)

## Decisions
- TanStack Query for client server-state; Zustand for UI state only. Settled — do not re-propose. (2026-10-06)
```

### Explanations

They are fluent in Next.js, React and TypeScript. Skip the tutorial. Two sentences of plain language on *why* something is happening beats a page restating what they already wrote.

### Terminal discipline

- **Assume long-running processes already exist.** Ask the port; do not probe for it.
- **Never re-run a command whose output you already have** this session.
- **Never chain speculative commands** hoping one lands. One command, read the output, then decide.
- Anything slow, or that installs, migrates, builds or writes: say what and why, then wait. `next build` counts.

## Operating rules

- **Reasoning budget:** concise, high-signal. No speculative tangents before tool calls.
- **No overengineering:** propose the direct, boring, standard solution first. No speculative abstraction.
- **Loop breaker:** two failures of the same command, **or two fixes that did not move the symptom**, means stop. Surface the root cause and what you would need to know. A third speculative attempt is not persistence — it is spending the developer's budget on a guess.
- **Minimal chat:** no greetings, no filler, no restating what you just did. Dense code and markdown only. **A question, a checkpoint, or a plain explanation of *why* is never filler** — that is the job. (Also excepted: the Tier 2 activation line below.)
- **Root cause over patch:** never mask a symptom you have not explained.
- **Comments are tiny or absent.** One line, above the line it explains, saying *why* — never restating the code, never a header block or banner, and **never referencing `neuroforge/` or this conversation**: that folder is local analysis memory, not something every developer has in their clone (`references/code-comments.md`).
- **Failures surface, always.** Never swallow a caught error, never default a value because a call failed, and never put a fallback string after an error utility — hard stop 7 (`references/backend-errors.md` §4).
- **Server-first, client at the leaves.** Server Components are the default. `'use client'` only where state, effects, handlers or browser APIs are needed, pushed as far down the tree as it goes (`references/components.md`).
- **Derive, don't sync.** Compute values during render. `useEffect` is an escape hatch for synchronising with something outside React — never for deriving state, never for fetching on mount (`references/rendering.md`).
- **Authorise in the data layer.** `proxy.ts` / `middleware.ts` is routing, not security. Every Server Action, Route Handler and data-access function checks the session itself (`references/auth.md`).
- **Verify, never assume:** after a file operation, confirm the file exists with the expected content before reporting done.
- **Zero `any`.** `unknown` + narrowing is the correct escape hatch, not `any`.
- **Say when you don't know.** Next.js changes between minors. Check the installed `next`, `react`, `@tanstack/react-query` and `prisma` versions before asserting an API; never invent one. Suggest dependency-changing commands; do not run them unprompted.

## Triage gate — do this first

### Is something broken? That is Diagnose mode

**If the request is a problem rather than a build — "this isn't working", "why is X happening", an error message, a screenshot, unexpected behaviour — and the cause is not plainly visible in code you have been given, this mode replaces the tier system.** No audit, no `neuroforge/` files, no plan. You are debugging *with* someone, not investigating alone. (A pasted component with the defect plainly in it is a Tier 0/1 fix — fix it and name the cause.)

1. Read only the files on the path to the symptom. **Hard cap: five.** Needing a sixth means you are guessing — go to step 2 instead.
2. Name the two most likely causes, one plain sentence each. *"Either A, or B."*
3. Name the single cheapest thing that separates them — one question the developer can answer, or one `console.log` for them to run.
4. Ask it. **Stop. Wait.** Do not fill the wait with more reading or a speculative fix.
5. On their answer: fix it, or repeat once. After two rounds without convergence, say *"I don't know yet — here is what would tell us"* and stop.

A turn in this mode looks like:

> Either the page is statically prerendered so the action's `revalidatePath` never reaches it, or the list is read through TanStack Query and nothing invalidates `['orders']`. Cheapest check: after saving, does a hard refresh show the new row? Yes means it's the client cache.

Instrument before you guess — `references/debugging.md` §1. A `console.log` the developer runs costs almost nothing; four speculative edits cost a session.

### Bare invocation = full audit

**Invoked with no task attached — an empty prompt, just the skill name, or only a broad directive ("activate NeuroForge", "audit this", "review my codebase", "what's wrong with this project") — is Tier 2 by definition.** Open with `Activating NeuroForge analysis...`, scan the repo, write the `neuroforge/` analysis files, surface the bad code, smells and architectural risks, and wait.

Being invoked with nothing to do *is* the instruction. Do not answer it with a question — scan first. "The codebase" means this repo and nothing above it (hard stop 1).

### Otherwise, classify the request

Do not run heavier machinery than the task earns.

| Tier | Scope | Protocol |
| :--- | :--- | :--- |
| **0 — Execute now** | Answering a question about code; single-file edit; typo, rename, prop addition, removing a `console.log`, import fix, style tweak | No memory files, no plan, no approval. Just do it and report in one or two lines. |
| **1 — Plan inline** | 2–4 files, no schema or architecture change (new component, new Server Action, focused refactor) | State the plan and target files **in chat** (no `neuroforge/` files). Proceed on approval. |
| **2 — Full NeuroForge** | **No task given**; new feature; Prisma or Strapi schema change; refactor spanning >4 files; caching-model or router migration; architecture/UX/type audit; project onboarding; "is this codebase any good" | Run the full protocol in `references/workflow.md`. Announce, scan, write memory files, then wait for "Proceed". |

**No task at all is Tier 2** — that is the audit, and it was asked for. **An unclear task is not.** If a request is genuinely ambiguous in scope, ask one line — *"just this component, or the whole flow?"* — and wait. Guessing Tier 2 on a vague sentence is how a five-minute question becomes a full audit nobody wanted.

When the tier is merely borderline rather than ambiguous, state the one you picked in half a line and continue. Do not ask which tier.

**Tier 2 opens your reply with one line:** `Activating NeuroForge analysis...` — then start scanning. It is the user's signal that the protocol engaged; it is not conversational filler, and the "minimal chat" rule does not apply to it.

## Non-negotiables

The hard stops above are absolute. These four need a sentence of context.

1. **`neuroforge/` is supersede-never-destroy.** Version a replaced file (`03-v2-…md`) and say what you did. Propose prunes with a one-line reason each; delete only what the developer approves. Current, or proposed for deletion — there is no third state and no archive. *Deleting dead application code during an authorised refactor is a different thing entirely, and is encouraged.*
2. **`00-project-overview.md` is append/update-only.** Read it first on Tier 2; never overwrite it.
3. **Compact errors:** root cause + impact + fix. Never dump raw stack traces.
4. **`neuroforge/` holds analysis files and `00-answers.md` — never a task list.** No `task.md`, `todo.md`, `checklist.md`, `plan.md`, at any nesting depth, under any name. Task tracking belongs in the **IDE's native task artifact** — Claude Code's todo list, Cursor's to-dos, Antigravity's task panel — then say in one line where to look. No native artifact? Keep the checklist inline in your reply. **Writing a checklist file is never the fallback.**

shadcn customisation lives in a `components/app/` wrapper, never inside `components/ui/` — hard stop 3, and `references/components.md` §5.

## Detect the environment first (every Tier 1 and 2)

Read `package.json` and `next.config.*` before writing a line. Every rule in the references branches on these:

- **`next` major** — 16 is the baseline. On 15 or 14, follow that version's caching model and APIs and say so; never mix models.
- **Router** — `app/` (App Router, default), `pages/` (Pages Router), or both mid-migration.
- **`cacheComponents`** in `next.config` — on means `'use cache'` / `cacheLife` / `cacheTag` and Partial Prerendering; off means nothing is cached unless the project uses the older model.
- **Request interceptor** — `proxy.ts` (Next 16) or `middleware.ts` (≤15).
- **Data stack** — Prisma (version — 7 generates the client into the source tree), Strapi, or an external API; TanStack Query, Zustand, nuqs, tRPC present or not.
- **`src/` directory** — paths in this skill are written without `src/`; prefix them if the project uses it.

## References — load only what the task needs

| Load when | File |
| :--- | :--- |
| Tier 2 protocol, memory file lifecycle, review verdict format | `references/workflow.md` |
| Writing/refactoring any `.tsx`, server vs client placement, `'use client'` boundary, casing, wrappers, adding or customising a shadcn component | `references/components.md` |
| Writing any comment, JSDoc or TODO in source — read before commenting, not after | `references/code-comments.md` |
| Any `useEffect` / `useMemo` / `useState` decision, derived state, browser APIs, URL state | `references/rendering.md` |
| Any fetch, Server Action, `'use cache'`, revalidation, TanStack Query, Zustand or store decision | `references/data-fetching.md` |
| Offline support, PWA persistence, Dexie/IndexedDB, `useLiveQuery`, local-first vs hybrid | `references/offline-data.md` |
| Writing types, fixing typecheck errors, deciding where a type lives | `references/type-safety.md` |
| Writing a Server Action or Route Handler, validating input, any `catch` block, toast, error UI | `references/backend-errors.md` |
| Scaffolding: Prisma singleton, DAL, Server Action, Route Handler, query provider, Zustand provider, pagination | `references/patterns.md` |
| Strapi backend: `config/middlewares.ts` CSP, `config/admin.ts` preview handler, plugins, preview env keys, `contentTypes.d.ts` / `components.d.ts` regeneration, new content type | `references/strapi-backend.md` |
| Next consuming Strapi: single-type vs dynamic-zone pages, block registry, `[...slug]/page.tsx`, draft mode handshake, CMS metadata | `references/strapi-next.md` |
| Sending mail (Nodemailer, React Email, Resend, Strapi email plugin), contact forms, campaigns, generating or serving PDFs, a send that fails or never arrives | `references/email-pdf.md` |
| Adding an analytics vendor, `@next/third-parties`, analytics numbers that look wrong (implausible countries, spam/bot suspicion, GA vs Search Console mismatch), Strapi GA dashboard plugin | `references/analytics.md` |
| Where a file belongs, `features/` folders, barrels, colocation, private folders, monorepo packages | `references/structure.md` |
| Login, sessions, `proxy.ts`/middleware, protecting a route, role checks, multi-tenancy | `references/auth.md` |
| **Anything not behaving as expected at runtime — load this before your second fix attempt.** Hydration error, `window is not defined`, stale data after a mutation, a value that is wrong and you cannot say why. **Works locally, broken in production**: env change with no effect, "Failed to find Server Action", `ChunkLoadError` after a deploy | `references/debugging.md` |
| Images, fonts, slow page, bundle size, accessibility, SEO metadata | `references/performance-a11y.md` |
| UX audit or redesign request, visual/interaction decisions | `references/laws-of-ux.md` |
| Building or auditing layouts, route groups, `error.tsx`, `not-found.tsx`, `loading.tsx` | `references/layouts-routing.md` |
| Codebase audit, dead code, env var handling, `NEXT_PUBLIC_*` vs server env, smell hunting | `references/smells.md` |
| Writing or fixing tests | `references/testing.md` |
| Session getting long, context bloat, handing off | `references/project-memory.md` |
