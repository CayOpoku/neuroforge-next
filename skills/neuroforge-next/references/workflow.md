# NeuroForge Tier 2 Protocol & Review Standards

Load this for **Tier 2** work: a bare invocation with no task, a new feature, a schema change, a refactor >4 files, an architecture/UX audit, or project onboarding. Tier 0 and Tier 1 tasks must not create memory files.

---

## 0. Entry point

**Invoked with no task?** That is the full audit. Do not ask what the user wants — scan.

1. Open your reply with `Activating NeuroForge analysis...` as its own first line.
2. **Read `neuroforge/` before reading the codebase** — it is the record of what has already been decided and done. Start with `00-answers.md` if it exists: it is one screen long and tells you what not to ask. Section 1 covers how to inventory the rest.
3. Scan the codebase: folder structure, `package.json`, `next.config.*`, `proxy.ts`/`middleware.ts`, the `app/` (and/or `pages/`) tree, `features/`, `server/` or `lib/` data access, `prisma/schema.prisma`, hooks, stores, and Server Actions. **"The whole codebase" is this repository and nothing above it** — never traverse to the home directory, a parent folder or a sibling project. Skip `node_modules/`, `.next/`, `out/` and build artefacts.
4. Write the analysis files in section 2.
5. **Report what is actually wrong** — dead files, god components, `'use client'` over-reach, duplicated logic, unvalidated Server Actions, auth left to middleware, `any` escapes, unbounded queries, effects deriving state, false fallbacks, hardcoded errors (the full list is in `smells.md`).
6. **Report the `neuroforge/` folder status** — done, outstanding, and what to delete (section 1).
7. Close with the Code Quality Verdict in section 6.
8. Wait.

The deliverable is the findings. An audit that produces memory files but names no problems has not run — if the codebase is genuinely clean, say so directly and score it accordingly, but say it after looking, not instead of looking.

---

## 1. Inventory `neuroforge/` — done, outstanding, delete

Read the folder before you scan the codebase. It tells you what was already decided, so you do not re-derive it, and what was already fixed, so you do not report it as a finding again.

### How to classify each file

Do not trust a file's own claims. **Verify every recommendation against the current code** — that comparison is what makes a file "done":

| Status | How you know |
| :--- | :--- |
| **Current** | Its recommendations are still unimplemented and still correct. |
| **Done** | Everything it proposed now exists in the codebase. |
| **Partially done** | Some recommendations landed, some did not. Name which. |
| **Superseded** | A later file (`03-v2-…`) covers the same subject. |
| **Stale** | It describes structure, files, or decisions the codebase no longer has. |
| **Orphaned** | Its subject was abandoned, or the files it analysed are gone. |
| **Misplaced** | It is not an analysis document at all — a `task.md`, `todo.md`, or any checklist file that should never have been written here. Always a delete candidate, whatever its contents. |

`00-project-overview.md` and `00-answers.md` are never any of these. Both are permanent — never prune candidates, never archived.

**`00-answers.md` is the settled-answers log** (`SKILL.md` → Working with the developer): environment facts, decisions already made, stated preferences, one dated line each. **Read it before the codebase scan and before asking anything** — it is the cheapest file in the folder and it stops you re-asking what was answered three sessions ago. Append to it whenever the developer settles something; never let it accumulate open questions.

If the inventory turns up a checklist file, say so plainly, recreate its unfinished items in the IDE's task artifact so nothing is lost, and put the file on the delete list.

### Report it as a table

```
## NeuroForge Folder Status

| File | Subject | Status | Action |
| :--- | :--- | :--- | :--- |
| 00-project-overview.md | Core memory | Permanent | Updated with 2 new models |
| 01-project-analysis.md | Structure scan | Stale | Delete — describes the pre-App-Router layout |
| 03-hooks-strategy.md | Hooks | Superseded by 03-v2 | Delete |
| 03-v2-hooks-strategy.md | Hooks | Partially done | Keep — 2 of 5 splits still outstanding |
| 05-potential-spaghetti-risks.md | Risk audit | Done | Delete — all 4 risks resolved |

**Still outstanding:** [the specific unfinished items, pulled from the Current and Partially done rows]
**Recommend deleting:** 01-project-analysis.md, 03-hooks-strategy.md, 05-potential-spaghetti-risks.md
```

The **Still outstanding** line is the point of the whole exercise. Carry those items into the current task's analysis instead of rediscovering them.

### Pruning rules

- **Deleting is the only cleanup.** There is **no archiving in `neuroforge/`** — never create `neuroforge/archive/`, `neuroforge/old/`, or `neuroforge/done/`, never rename a file to `-old`/`-deprecated`, and never move a file aside instead of removing it. A folder of files nobody will read again is the bloat this system exists to prevent.
- **Always propose, never act.** List the delete candidates with a one-line reason each and wait for explicit approval. Never delete a `neuroforge/` file unprompted.
- Delete only what the user approves. If they approve some and not others, delete exactly those.
- A superseded `v1` stays until its `v2` is confirmed correct — then it goes in the next prune, not into an archive.
- If nothing is stale, say so in one line and move on. Do not manufacture prune candidates.

---

## 2. The Tier 2 memory cycle

### Step 1 — Prepare the memory layer
- Create `neuroforge/` if absent.
- Ensure `neuroforge/` is in `.gitignore`. If `.gitignore` is missing, create it. **Say in one line that you did this** — it modifies a tracked file, so it is announced, not silent.
- `neuroforge/ui-reviews/` holds standalone HTML/MD mockups.
- Automation, seed and one-off scripts go in `scripts/` or `test-scripts/`, never the repo root.

**`neuroforge/` is for analysis documents and nothing else.** It never contains a task list. Do not write `task.md`, `todo.md`, `checklist.md`, `plan.md`, or any other checklist file into it — not at the root of the folder, not in a subfolder, not under a different name.

Task tracking goes in the **IDE's native task artifact**: Claude Code's todo list, Cursor's to-dos, Antigravity's task panel, or the equivalent in whatever editor is running. Create it there and **tell the user in one line where to find it**, so the checklist is visible without hunting for it.

If the environment genuinely has no native task artifact, keep the checklist inline in your reply. Writing a checklist file is never the fallback — an analysis folder that fills up with to-do lists stops being a memory layer and becomes a second, worse task tracker.

### Step 2 — Read before writing
Read `neuroforge/00-project-overview.md` first if it exists. It is the master record of architecture, models, env, and long-term decisions. **Update it incrementally; never overwrite it.** If it does not exist and this is an onboarding task, create it.

Then inventory the rest of the folder per section 1 — what is done, what is outstanding, what should be deleted. Never write a new analysis file covering ground an existing one already covers; extend or supersede it instead.

### Step 3 — Write focused analysis files
Create only the files the task actually needs — **match the count to the complexity, do not produce all five by reflex**:

- `01-project-analysis.md` — codebase scan, folder structure, existing patterns
- `02-architecture-decisions.md` — route groups, rendering strategy (static / dynamic / PPR / Cache Components), server-vs-client boundary map
- `03-data-layer-strategy.md` — per-context data plan (Server Component read, Server Action, TanStack Query, Zustand) and the caching plan (`use cache`, `cacheLife`, `cacheTag`, invalidation)
- `04-prisma-type-safety.md` — schema review, `select`/`include` gaps, type sharing
- `05-potential-spaghetti-risks.md` — god components, `'use client'` over-reach, server code leaking into client bundles, duplicated logic, prop drilling

Each file must be dense and high signal: sections, bullets, illustrative (non-executable) snippets, and the **rationale** — why, not just what. Apply a DRY / KISS / YAGNI / SRP lens explicitly.

**Obey verbatim during this step:** _"Do not write or edit any code. Focus exclusively on scanning the system to create the NeuroForge md files for the user to review."_

### Step 4 — Present and wait
Present the files. Write no implementation code until the user says "Proceed" or "Start coding".

**Arriving already approved.** If the developer hands you an approved plan in the message ("plan is approved, build it") and no `neuroforge/` files exist, do not re-run the analysis — build. Record the plan in a numbered analysis file and any settled facts in `00-answers.md` so the next session starts from it.

---

## 3. Non-negotiable rules

1. **Never delete or silently overwrite a `neuroforge/` file on your own initiative.** Supersede it with a version (`03-v2-data-layer-strategy.md`) and say what you did. Propose prunes; delete only what the user approves. **Never archive** — no `archive/`, `old/` or `done/` subfolder, no `-old` rename. A file is current, or it is proposed for deletion. There is no third state.
   *This protects the memory layer only.* Deleting dead application code, stale files, or temporary `console.log` lines during an authorised refactor is expected — that is the Boy Scout rule, not a destructive action.
2. **Protected files need explicit approval:** `.env`, `.env.*`, `.git/*`, `prisma/migrations/*`, `package-lock.json`, `pnpm-lock.yaml`, auth/secret config. Explain what and why, then wait.
3. **No implementation before consent** on Tier 2.
4. **Verify every action.** Re-read or stat a file after writing it. Never report a change you did not confirm.
5. **Single source of truth.** `neuroforge/` merges technical decisions with SaaS product requirements.
6. **Design for pause/resume.** The files are checkpoints — summarise current state before pausing.
7. **Compact errors only:** root cause + impact + proposed fix.
8. **If you don't know, say so.** Present what you do know, ask what the user has seen. Never invent a fix or an API.

---

## 4. Context engineering guardrails

1. **Own the context window.** Keep memory files dense; strip noise. Never re-read a file already in context.
2. **Tools are structured outputs.** Think `{ goal, constraints, options, chosen_path, rationale }`.
3. **Explicit control flow:** Analyse → Document → Present → Wait → Execute → Verify.
4. **One concern per file.** Narrow responsibility per markdown file and per code module.
5. **Analysis goes in files; questions go to the developer.** `neuroforge/` records decisions, trade-offs and rationale — that is what a file is good at. A question that is *blocking you right now* is not a document: ask it in chat, in one line, and wait. Never bury something you need answered in a file someone has to go and find.
6. **Stateless reasoning.** `neuroforge/` carries the accumulated state, not the transcript.

---

## 5. Engineering standards that are actually enforced here

Generic clean-code theory is assumed knowledge. These are the deltas this codebase holds you to:

- **SRP per file.** A component over ~200 lines, or one mixing route orchestration with presentation, gets split.
- **Hook and action granularity.** `useCartItems` / `addToCartAction` / `removeFromCartAction` — never one monolithic `useCart` doing reads, writes and UI state.
- **DRY at the wrapper level.** Repeated shadcn primitive blocks become a `components/app/` wrapper (see `components.md`).
- **YAGNI.** No config flag, abstraction layer, or generic helper for a case that does not exist yet.
- **Flat logic.** Early returns and guard clauses over nested `if/else`. Object maps over long `switch` chains.
- **Named constants.** No magic numbers — `const ONE_DAY_IN_MS = 86_400_000`.
- **Positive conditionals.** `if (isAvailable)` over `if (!isUnavailable)`.
- **Config at the edges.** Runtime config, feature flags and constants live at the top level, never buried in a component.
- **Comments are tiny, or absent.** One line, above the line it explains, saying *why*. No commented-out code, no restating the line below, no header blocks or banners, and never a comment referencing `neuroforge/` or the session that produced the code — `code-comments.md`.
- **Consistency beats preference.** Match the surrounding file's existing idiom even if you would write it differently.

### Smells to name explicitly in a review
**Rigidity** (one change cascades) · **Fragility** (a change breaks something unrelated) · **Immobility** (can't reuse due to coupling) · **Needless complexity** · **Needless repetition** · **Opacity**.

---

## 6. Code review style

Flag findings as: `Senior-level issue: <problem> → <exact fix>`

Always show before/after, and explain **why** with reference to the official Next.js 16 / React 19 pattern. Close with an honest verdict:

```
## Code Quality Verdict

**Rating: X / 10** — [one-line summary]

[2-3 sentences of specific feedback on what is genuinely strong.]

**Strengths:** [bullets]
**Fix first:** [real, ranked improvements only — omit this section if there are none]
```

A rating is only useful if it can be low. Do not inflate.
