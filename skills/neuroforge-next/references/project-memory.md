# Project Memory & Session Management

Applies to **Tier 2** work. Tier 0 and Tier 1 tasks create no memory files — see the triage gate in `SKILL.md`.

Handoff specification: [skills.sh handoff standard](https://www.skills.sh/mattpocock/skills/handoff)

---

## 1. `00-project-overview.md` — the core record

- **Never overwrite it.** It is the master record of architecture, database models, environment configuration, and long-term decisions.
- **Read it first** on any Tier 2 task, before scanning the codebase — it usually answers questions a scan would cost thousands of tokens to re-derive.
- **Update it incrementally** whenever a new model, route group, layer, or architectural pattern is introduced. Append; do not rewrite history.
- If it does not exist and the task is onboarding, create it.

## 2. Task pre-analysis files

- Task-specific files (`01-project-analysis.md`, `02-architecture-decisions.md`, …) may be created or overwritten freely — they are scoped to the current task.
- Create only the ones the task needs. Producing five files for a two-file change is the exact token waste this system exists to prevent.
- **These are analysis documents.** They record findings, decisions, and rationale — never a checklist of steps to tick off.

### No task files in `neuroforge/`

Never create `task.md`, `todo.md`, `checklist.md`, `plan.md`, or any other checklist file inside `neuroforge/` — at any depth, under any name. This has gone wrong before; the folder is for analysis only.

Task tracking belongs in the **IDE's native task artifact** — Claude Code's todo list, Cursor's to-dos, Antigravity's task panel, or the running editor's equivalent. Create the checklist there, then point the user to it in one line.

No native task artifact available? Keep the checklist inline in the reply. A checklist file is never the fallback.

---

## 3. Large tasks: write a script instead of reading

For a massive refactor, a multi-file scan, or a bulk migration, do not read thousands of lines into context.

Write a Node (or `tsx`) script into `scripts/` or `test-scripts/` that performs the scan or transformation, present it for review, and let the user run it. The script is cheaper than the context, reviewable before it runs, and repeatable.

---

## 4. Token burn and handoff

Watch session length. When context is getting heavy, say so and offer a handoff document rather than degrading quietly.

A handoff document:

1. **Saves to the OS temp directory** (`%TEMP%` on Windows, `/tmp` on POSIX) — never inside the repo.
2. **Includes a "Suggested Skills" section** naming the exact skills a fresh agent should invoke — `neuroforge-next` plus any companion skill this work depends on (e.g. `neuroforge-threejs` for a 3D section, or a shadcn/TanStack skill the session leaned on), so the next agent does not re-derive which tools it needs.
3. **References, never duplicates.** Point at files, commits, and `neuroforge/` paths instead of pasting their content.
4. **Redacts everything sensitive** — secrets, keys, tokens, PII.
5. **States what the next session is for**, incorporating whatever the user says the next focus is.

---

## 5. Pruning — delete, never archive

`neuroforge/` accumulates finished plans, old UI reviews, and superseded audits. Left alone it becomes the context bloat this system exists to prevent.

**Inventory it on every Tier 2 entry** (the classification table and report format are in `workflow.md` section 1): read each file, verify its recommendations against the current code, and mark it Current / Done / Partially done / Superseded / Stale / Orphaned. Report what is **still outstanding** — that is the part that earns the read — and list what should go.

**There is no archiving.** Never create `neuroforge/archive/`, `neuroforge/old/`, or `neuroforge/done/`. Never rename a file to `-old` or `-deprecated`. Never move a file aside instead of deleting it. A memory file is either current or a delete candidate; a folder of files nobody will open again is bloat wearing a tidier name.

**Always ask before deleting.** Propose the list with a one-line reason per file and wait for explicit approval. Delete exactly what the user approves, nothing more. Never delete a memory file silently.

`00-project-overview.md` is permanent and is never a prune candidate.

When a file is superseded, version it (`03-v2-data-layer-strategy.md`) and leave the original until the `v2` is confirmed correct — then it joins the next prune list.
