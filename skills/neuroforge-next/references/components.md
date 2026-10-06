# Components: Server/Client Boundaries, Structure & Reusability

Paths are written without `src/`; prefix them if the project uses a `src/` directory.

---

## 1. Server or client — decide first

Every component starts as a **Server Component**. Add `'use client'` only when it needs at least one of:

- state (`useState`, `useReducer`, `useActionState`, `useOptimistic`)
- effects (`useEffect`, `useLayoutEffect`)
- event handlers (`onClick`, `onChange`, …)
- browser-only APIs (`window`, `localStorage`, `IntersectionObserver`)
- a client-only library (most animation, interactive charts, form libraries, TanStack Query hooks)

None apply? It stays a Server Component: zero JS shipped for it, it can be `async`, and it can read data and secrets directly.

### The `'use client'` boundary checklist

- **It marks a boundary, not a file type.** Every module a Client Component imports is bundled for the browser. A heavy import in a client file ships to every visitor.
- **Never lift `'use client'` to a `page.tsx` or `layout.tsx`** to make one widget interactive. Extract the widget into its own client file and keep the page on the server.
- **Server Components can be passed into Client Components as `children` or props.** That is how a client shell (a tab set, a modal, a provider) wraps server-rendered content without becoming its owner.
- **Props crossing server → client must be serialisable**: plain objects, arrays, strings, numbers, `Date`, `Map`/`Set`, promises (consumed with `use()`), and Server Actions. Not class instances, not arbitrary functions, not Prisma results with `Decimal`/`BigInt` you haven't converted.
- **Pass only what the client needs.** A Server Component handing a whole user row to a client avatar ships the email and role to the browser. Select the two fields.
- **`import 'server-only'`** at the top of any module holding secrets, DB access, or privileged logic. A client import then fails the build instead of leaking.
- Never read a non-`NEXT_PUBLIC_` env var in a Client Component — it is `undefined` there, and the "fix" people reach for is renaming it `NEXT_PUBLIC_`, which publishes the secret.

---

## 2. Styling: Tailwind is the house style

This stack is Tailwind + shadcn/ui. **Style with utility classes.** No CSS module or styled-component per component by default — two places to check for every visual rule.

A CSS file is justified only for what utilities genuinely cannot express: keyframes, complex `::before`/`::after` art, third-party widget overrides. Say why in a one-line comment.

Design tokens live in the Tailwind theme and the shadcn CSS variables — never a hardcoded hex value in a component. Conditional classes go through `cn()` (`lib/utils.ts`, shadcn's helper), never string concatenation.

---

## 3. Inside a component — a predictable order

1. Props type and destructuring
2. Hooks (state, context, queries, router)
3. Derived values — computed during render (`rendering.md`)
4. Handlers
5. Effects (client only, and rare)
6. Return

```tsx
type InvoiceRowProps = { invoice: InvoiceListItem; onSelect: (id: string) => void }

export function InvoiceRow({ invoice, onSelect }: InvoiceRowProps) {
  const isOverdue = invoice.status === 'OPEN' && invoice.dueAt < new Date()

  return (
    <tr className={cn(isOverdue && 'text-destructive')} onClick={() => onSelect(invoice.id)}>
      <td>{invoice.number}</td>
    </tr>
  )
}
```

Named exports for components; `export default` only where Next.js requires it (`page.tsx`, `layout.tsx`, `error.tsx`, `not-found.tsx`, `loading.tsx`, `template.tsx`, `default.tsx`, `route` files don't default-export).

---

## 4. Reusable shadcn wrappers (`components/app/`)

### Write once, use everywhere

Never paste twenty lines of raw shadcn primitives into a page. Wrap them:

- `components/app/app-button.tsx` → `<AppButton>`
- `components/app/app-dialog.tsx` → `<AppDialog>`
- `components/app/app-data-table.tsx` → `<AppDataTable>`
- `components/app/app-form-field.tsx` → `<AppFormField>`

Expose props, variants, sizes and slots (`children`, named render props) on the wrapper; hide the repetitive internal markup. Forward the rest so consumers can still reach the underlying element:

```tsx
import type { ComponentProps } from 'react'
import { Button } from '@/components/ui/button'
import { Loader2 } from 'lucide-react'

// ComponentProps, not ButtonProps: newer shadcn styles no longer export a props type
type AppButtonProps = ComponentProps<typeof Button> & { loading?: boolean }

export function AppButton({ loading = false, disabled, children, ...rest }: AppButtonProps) {
  return (
    <Button disabled={disabled || loading} aria-busy={loading} {...rest}>
      {loading ? <Loader2 className="mr-2 size-4 animate-spin" aria-hidden /> : null}
      {children}
    </Button>
  )
}
```

**Before writing any new UI element, inspect `components/app/` first.** Reuse beats a second wrapper for the same primitive.

---

## 5. shadcn primitives come from the CLI — never from your keyboard

`components/ui/` is **generated output**. There is exactly one way a primitive enters this project:

```bash
npx shadcn@latest add pagination
npx shadcn@latest add dialog table sheet   # several in one call
```

**Never hand-write a primitive, never paste one in from the docs, another project or memory, and never edit one in place.** "It is only a pagination component" is exactly the case this rule exists for.

### Why the CLI is not optional

- It installs the registry version matching this project's `components.json`, alias paths, Tailwind version and Radix packages. A hand-written file targets whatever the author remembered.
- It pulls transitive primitives and npm dependencies (`form` brings `react-hook-form` and its resolver).
- It writes the exact upstream file, so `npx shadcn@latest diff` and future upgrades keep working. A lookalike is a silent fork.
- Upstream primitives carry keyboard handling, focus management and ARIA wiring that a from-memory reproduction quietly drops (`performance-a11y.md`).

### Procedure when a primitive is missing

1. Look in `components/ui/` first — it may already be installed.
2. If not, **surface the exact `npx shadcn@latest add …` command and stop.** It changes dependencies, so suggest it; do not run it unprompted.
3. Once the user has run it, build the `components/app/` wrapper (§4) on top and put every customisation there.

Do not route around this with a copy under another name. If the component genuinely is not in the registry, say so — then it is a custom component, it goes in a domain folder, it is built on Radix primitives, and it still never lands in `ui/`.

### When the `add` command fails

**A failing CLI is a bug to fix, never a licence to hand-write the component.** Stop, report the exact error, and work the cause with the user:

- No `components.json` → never initialised: `npx shadcn@latest init`.
- "Component not found" → wrong name or wrong registry.
- Alias or path errors → `components.json` aliases do not match `tsconfig.json` `paths` (common with `src/`).
- Tailwind v3 vs v4 mismatch → the registry style and the installed Tailwind disagree; fix the setup, don't hand-port.
- Network, proxy or peer-dependency errors → fix the environment or the dependency.

Leaving the component uninstalled is the correct outcome; a hand-written `ui/` file is not.

---

## 6. Folder organisation

Domain-driven, never page-bound. `home-hero.tsx` and `about-card.tsx` are the wrong shape — the moment a second page needs them, the name lies.

```
components/
  app/          -> reusable wrappers (AppButton, AppDialog, AppDataTable)
  ui/           -> raw shadcn primitives ONLY; never put custom components here
  content/      -> content display widgets (ContentCard, ContentList)
  form/         -> form inputs (FormOtp, FormSelect)
features/
  billing/components/   -> components owned by one domain (structure.md)
```

**Three hard rules for `ui/`:** a custom component never lands there; never edit a file there (if a change cannot live in a wrapper, tell the user which file and why); never write one there by hand.

**File names are kebab-case** (`app-data-table.tsx`, `form-otp.tsx`) — shadcn's convention, and it avoids case-sensitivity bugs between macOS/Windows and Linux CI. **Component names are PascalCase.** Route-segment-only components can be colocated in a private folder (`app/(app)/billing/_components/`) — see `structure.md`.

---

## 7. Component size and responsibility

- Over ~200 lines, or mixing data loading with presentation: split it. A Server Component that fetches and a Client Component that renders interactively are usually two files.
- JSX stays declarative. Anything past a ternary moves to a derived const or a small component.
- **Count the effects.** A component with several `useEffect` blocks is usually syncing state it should derive (`rendering.md`).
- Data access and business rules live in `features/<x>/server/` and hooks; the component renders and calls.
- More than ~7 props usually means two components, or one that should take an object.
- Prop drilling past two levels: composition (`children`), context, or a store (`data-fetching.md` §5).

---

## 8. External templates (`templates/`)

Never embed HTML email bodies, PDF layouts, or large static data structures inside a page component or Route Handler. Email templates are React Email components in `templates/email/`; PDF documents in `templates/pdf/` (`email-pdf.md`); large static data in `data/`.

---

## 9. Image placeholders

```tsx
<img src="https://placehold.co/1500x1500" alt="Placeholder" className="h-full w-full object-cover" />
```

Match the dimensions to the container. Placeholders are for drafts only — replace them with `next/image` and real assets before shipping (`performance-a11y.md`), and add the host to `images.remotePatterns` if a remote source is kept.
