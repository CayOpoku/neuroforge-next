# Laws of UX & Interface Standards

Reference: [Laws of UX](https://lawsofux.com/)

---

## 1. UX audit protocol

When asked to improve, audit, or redesign UX, run a short interview before proposing anything. Challenge the assumptions rather than restyling what is already there.

Ask about: who the user is and what they are trying to finish; what the primary action on this screen is; what happens when it fails; what the user sees on first load with no data; where they came from and where they go next.

Then apply:

- **Fitts's Law** — primary actions get large targets placed near where the cursor already is. Destructive actions do not sit next to primary ones.
- **Hick's Law** — every additional choice on a critical path costs time. Collapse or defer secondary options.
- **Jakob's Law** — users expect your app to behave like the apps they already use. Do not reinvent a date picker, a table filter, or a login flow without a reason you can state.
- **Miller's Law** — chunk information into groups of roughly 5–9. A 30-field form is several forms.
- **Peak-End Rule** — success and recovery states carry disproportionate weight. A clear confirmation is worth more than a polished idle state.
- **Postel's Law** — accept input liberally (trim whitespace, accept any phone format), emit it strictly.
- **Doherty Threshold** — feedback under 400ms. Past that, show a skeleton, not a spinner in an empty page.
- **Aesthetic-Usability Effect** — a polished interface is perceived as more usable, which buys goodwill but hides real friction in testing. Do not let it substitute for fixing the flow.

---

## 2. Mandatory UI rules

- **No emojis in the UI.** Never render a raw emoji in JSX. Use SVG icons — `lucide-react` (shadcn's default) or the project's icon set. Emojis render inconsistently across platforms, do not inherit colour, and read unpredictably to screen readers.
- **Light mode is the default.** Never force dark mode or ship it as the default unless the user explicitly asks. Support both; default to light.
- **Four states, always.** Every data-bound view handles loading, empty, error, and success. An empty state that looks identical to a failed one is a bug.
- **No presumptuous fallbacks.** If a status fails to load, do not render "Pending" or "Active". Show the failure. Never make a decision on the user's behalf through a silent default.
- **Destructive actions require intent** — a confirmation naming what will be deleted, and never a default-focused destructive button.
- **Never block on an avoidable modal.** If the choice can be deferred or undone, prefer an inline affordance and an undo.

---

## 3. Error visibility

- The user must never be left guessing what went wrong.
- Render the exact backend message (`backend-errors.md`). Structured validation errors go next to their fields, not flattened into a toast.
- An error message states what happened **and what to do next**. "Invalid card number" beats "Payment failed".
- Errors must be announced to assistive technology (`role="alert"`) — see `performance-a11y.md`.

---

## 4. Accessibility is part of UX, not a separate pass

Keyboard operability, focus management, labels, contrast, and motion preferences are covered in `performance-a11y.md`. A design that fails those is not finished, regardless of how it looks.
