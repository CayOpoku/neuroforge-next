# Code Comments

Comments are load-bearing or they are noise. This file sets the bar for every comment written in this codebase.

Adapted for Next.js/React from the `code-comments` skill (skills.sh). Where the two differ, this file wins — it is tuned for a convention-driven Next.js repo where the path (`app/(app)/orders/page.tsx`, `features/billing/actions/`) already says what a file is.

---

## 1. The bar

**Default: no comment.** Write the code so the comment is unnecessary — a better name, an early return, an extracted hook, a named constant. Only when the *why* genuinely cannot live in the code does a comment earn its place.

**The why test:** if the comment restates *what* the line does, delete it. If it explains *why this line and not the obvious one*, keep it.

```ts
// ✗ Restates the code
// Fetch the user's orders
const orders = await listOrdersForCurrentUser()

// ✓ Explains the non-obvious choice
// Page is in the key so back-navigation refetches instead of serving page 1's cache.
const { data } = useQuery({ queryKey: ['orders', page], queryFn: () => listOrders(page) })
```

---

## 2. Size and shape

- **One line.** Two only for a real trade-off with a real cost. Never a paragraph, never a banner, never an essay above a 10-line function.
- **Sentence case, plain language, no ceremony.** No `/** */` block where `//` says it.
- **Above the line it explains**, never trailing off the end of a long line.
- **No file-header blocks.** `features/orders/server/orders.ts` does not need a header announcing that it holds order data access. A single line is allowed only where the purpose is genuinely not inferable from the path — a block registry, a webhook handler bound to an external contract, a file whose shape is dictated by a third party.
- **No section banners** (`// ===== STATE =====`, `// --- handlers ---`). A file that needs signposting needs splitting — `workflow.md` §5.

---

## 3. Never reference the `neuroforge/` folder — or this session

`neuroforge/` is local analysis memory. It is not part of every developer's workflow, it may never be committed, and a teammate cloning the repo will not have it. A comment pointing into it is a dead link the day it is written.

```ts
// ✗ Dead reference — the reader does not have this file
// See neuroforge/03-orders-architecture.md for why we denormalise here.

// ✓ Points at something that travels with the code
// Totals are denormalised on Order — the dashboard aggregates 10k+ rows per tenant (#412).
```

**Comments may only reference things that ship with the repo:** a file path in the repo, a symbol name, an issue/PR id, a migration name, an official docs URL.

The *reasoning* lives in the analysis file; the *one-line why* lives in the code and must stand alone without it. Never copy paragraphs out of `neuroforge/` into a source file.

**Equally banned — narrating the session or your own work:**

```ts
// ✗ Added by the assistant
// ✗ Updated to fix the bug reported earlier
// ✗ As requested, moved this to a Server Action
// ✗ Step 3 of the refactor
// ✗ v2 — new version of the handler above
```

Git records who changed what and when. A comment that describes the *edit* rather than the *code* is stale the moment the next edit lands.

---

## 4. What actually earns a comment here

| Situation | Example |
| :--- | :--- |
| A `useEffect` where derivation would be expected | `// Effect, not derived: the editor is uncontrolled and must not re-render per keystroke.` |
| A client-only import | `// Chart lib reads window at import time — loaded with ssr: false.` |
| A deliberate `'use client'` higher than a leaf | `// Whole panel is client: drag state spans every child.` |
| A non-obvious cache decision | `// cacheLife('minutes'), not hours: prices change on the hour and staleness costs money.` |
| `updateTag` vs `revalidateTag` | `// revalidateTag, not updateTag: this runs from the CMS webhook, not a user's own save.` |
| A deliberate Prisma narrowing | `// select, not include — passwordHash must never leave this function.` |
| A wrapper deviating from the primitive | `// 44px target instead of the primitive's 36px: primary touch action (Fitts).` |
| A workaround, with its expiry | `// Workaround for vercel/next.js#12345 double render. Remove once we're on 16.2.` |
| A guarantee that justifies a non-null assertion | `// requireUser() redirected already; a null user here is a bug, not a state.` |
| `suppressHydrationWarning` | `// next-themes sets the class before hydration — the mismatch is intentional.` |

Everything on this list is one sentence a reader could not have recovered from the code.

---

## 5. What never earns one

1. Restating the line below it.
2. Commented-out code — git has it, delete it (`workflow.md` §5).
3. Values a constant already names: `// 5 minutes` above `STALE_AFTER_MS`. Unit translations (`1_048_576 // 1MB`) are fine.
4. Types TypeScript already states: `@param userId - the user id` on `userId: string`.
5. Obvious props, obvious getters, obvious one-line helpers.
6. Vague intent: `// TODO: refactor later`, `// clean this up`, `// might need this`.
7. Anything from §3 — `neuroforge/` paths, session narration, changelog lines.

---

## 6. JSDoc

Only on **exported** utilities, hooks, actions and data-access functions whose contract is not obvious from the signature. One sentence on what it guarantees, plus anything a caller would get wrong. Never a `@param` table that repeats the types.

```ts
/** Returns the backend's message verbatim. Never falsy — do not `||` a fallback onto it. */
export function getErrorMessage(error: unknown): string
```

```ts
/** Scoped to the session's organization — callers must not pass a client-supplied tenant id. */
export async function listInvoices(): Promise<InvoiceListItem[]>
```

---

## 7. JSX

Keep `{/* */}` rare. JSX that needs headings to be navigable is a component that needs splitting (`components.md` §7). The legitimate uses are a non-obvious structural constraint or an accessibility decision:

```tsx
{/* aria-live on the wrapper, not the row: screen readers miss per-row updates. */}
<div aria-live="polite">
```

---

## 8. TODO / HACK / FIXME

Actionable and traceable, or not written at all. Owner or issue id, and the condition that retires it.

```ts
// TODO(#412): move totals into the aggregate query once the reporting view lands.
// HACK: Safari 17 flexbox gap bug — drop when we stop supporting 17.x.
// FIXME(#488): rapid toggling races; the in-flight mutation is not cancelled.
```

No issue and no owner means it is not a TODO, it is litter.

---

## 9. Maintenance

- **A stale comment is a bug.** When you edit a line, re-read the comment above it — update it or delete it in the same edit.
- **Delete comments with the code they describe.** An orphaned comment above a rewritten block is confidently wrong.
- **Match the file.** If the surrounding code comments sparsely, comment sparsely (`workflow.md` §5, consistency).

---

## 10. Review checklist

Flag in an audit:

1. Header blocks or banners restating what the path already says.
2. Any comment naming `neuroforge/`, an analysis file, or the conversation that produced the code.
3. Comments describing the edit rather than the code (`// updated`, `// new`, `// as requested`).
4. Commented-out code.
5. JSDoc restating types, or `@param` tables on self-describing signatures.
6. TODOs with no owner and no issue.
7. Comments contradicting the code beneath them.
