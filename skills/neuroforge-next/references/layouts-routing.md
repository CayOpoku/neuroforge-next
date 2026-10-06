# Layouts, Routing & Page Architecture

App Router conventions. For a Pages Router project, see §5.

---

## 1. Essential SaaS route groups

Layouts are a primary architectural decision, not an afterthought. Route groups give each audience its own shell without changing the URL:

```
app/
  layout.tsx                 -> root: <html lang>, fonts, Providers — nothing audience-specific
  (marketing)/
    layout.tsx               -> navbar + footer
    page.tsx                 -> /
    pricing/page.tsx
  (auth)/
    layout.tsx               -> focused, distraction-free shell
    login/page.tsx
  (app)/
    layout.tsx               -> sidebar, user header, breadcrumbs
    dashboard/page.tsx
  not-found.tsx              -> app-wide 404
  global-error.tsx           -> catches errors in the root layout itself
```

Plus, per segment where it earns it:

- **`loading.tsx`** — instant skeleton while the segment's Server Components resolve. One per group is usually right; nest more only where a section has its own slow data.
- **`error.tsx`** — must be a **Client Component** (`'use client'`); receives `{ error, reset }`. Shows the real situation — "Something went wrong" for every failure is the false-fallback smell at page scale. In production `error.message` is redacted for server errors (`backend-errors.md` §2) — show a useful generic plus `error.digest` so support can find the log line.
- **`not-found.tsx`** — rendered by `notFound()`. Use it for missing resources, not a hand-rolled "not found" component with a 200 status.

---

## 2. Separation of concerns

- **`page.tsx`** is a thin route entry point: read `params`/`searchParams`, call the feature's data function, compose components. A page holding presentation markup or business logic is doing two jobs.
- **`layout.tsx`** owns persistent chrome. Layouts **persist across navigation and do not re-render** between child routes — great for a sidebar, wrong for anything page-specific, and **not a place to do auth checks** (`auth.md` §1).
- **`template.tsx`** is a layout that remounts on navigation — use it only when you need that (enter animations, per-page state reset).
- **Components** live in `components/` or the feature folder, never named after the page that first used them.

---

## 3. Routing rules

- **`params` and `searchParams` are untrusted input** (and Promises in Next 15+). Validate before use — a `[id]/page.tsx` that passes `params.id` straight to Prisma will happily query `undefined` or a non-UUID:

```tsx
const ParamsSchema = z.object({ id: z.string().uuid() })

export default async function OrderPage(props: PageProps<'/orders/[id]'>) {
  const parsed = ParamsSchema.safeParse(await props.params)
  if (!parsed.success) notFound()
  const order = await getOrderForCurrentUser(parsed.data.id)
  if (!order) notFound()
  return <OrderView order={order} />
}
```

- **`notFound()` before anything streams.** Call it at the top of the page (before a Suspense boundary renders content) so the response is a real 404. A not-found decided inside a streamed boundary arrives after a 200 has been sent.
- Protect routes in the data layer (`auth.md`); `proxy.ts` only redirects.
- Set the rendering strategy deliberately (`data-fetching.md` §2): marketing pages build to static shells; authed dashboards are dynamic and gain nothing from prerendering.
- Use `<Link>` for internal navigation — `<a href>` triggers a full reload and drops client state. `<Link>` prefetches in production; set `prefetch={false}` on links to heavy, rarely-visited pages in long lists.
- `generateStaticParams` for a known, bounded set of dynamic pages (blog posts, docs). Not for user data.
- **Parallel and intercepting routes** (`@modal`, `(.)photo/[id]`) for URL-addressable modals — powerful, and a maintenance cost. Reach for them only when the modal must be linkable and survive a refresh; otherwise a client dialog is simpler.

---

## 4. Metadata

Every public page exports `metadata` or `generateMetadata` (`performance-a11y.md` §6). Route groups can set a shared title template in their layout:

```ts
export const metadata: Metadata = { title: { template: '%s — Acme', default: 'Acme' } }
```

---

## 5. Pages Router (respect it where it exists)

- `getServerSideProps` / `getStaticProps` / `getStaticPaths` for data, `pages/api/*` for endpoints, `_app` / `_document` for the shell.
- Don't sprinkle App Router primitives (`'use cache'`, Server Actions, `notFound()` from `next/navigation`) into Pages Router files — they don't apply there.
- Mixed codebases are fine mid-migration. Keep the boundary explicit, record the migration direction in `02-architecture-decisions.md`, and recommend App Router for **new** surfaces without forcing a rewrite of working Pages code.
