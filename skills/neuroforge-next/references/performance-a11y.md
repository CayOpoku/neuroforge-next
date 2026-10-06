# Performance, Accessibility & SEO

---

## 1. Images

Never ship a raw `<img>` for content imagery. Use `next/image`:

```tsx
import Image from 'next/image'

<Image
  src="/hero.jpg"
  alt="Team reviewing the analytics dashboard"
  width={1200}
  height={630}
  sizes="(min-width: 1024px) 1200px, 100vw"
  preload            // LCP image only — `priority` on Next ≤15
/>
```

- **Always set `width`/`height`** (or `fill` inside a sized, `relative` parent). Missing dimensions cause layout shift — a Core Web Vitals penalty and visible jank.
- **The LCP image is preloaded and never lazy.** Everything below the fold stays on the default lazy loading. Check which prop the installed version uses (`preload` in 16, `priority` before) — don't guess.
- **`sizes` is mandatory with `fill` or responsive layouts.** Without it the browser downloads the largest variant on every phone.
- Remote hosts go in `images.remotePatterns` — specific hosts, never `**`. Next 16 also restricts `images.qualities` and requires `images.localPatterns` for local images with query strings — check the config before "fixing" a blocked image with a wildcard.
- `alt` describes the content's purpose. Decorative images get `alt=""`, never a missing attribute.

---

## 2. Fonts

`next/font` (`next/font/google` or `next/font/local`) self-hosts, subsets, and sets `font-display` with size-adjusted fallbacks — no layout shift, no third-party request. Load fonts once in the root layout and expose them as CSS variables for Tailwind. Never a `<link>` to Google Fonts.

---

## 3. Ship less JavaScript

- **Server Components first** — the biggest lever. A component that stays on the server ships nothing (`components.md` §1).
- **`'use client'` at the leaves**, and keep heavy libraries (charts, editors, maps) in the smallest client file that needs them.
- **`next/dynamic`** for client components below the fold, behind a tab, or inside a modal:

```tsx
const AnalyticsChart = dynamic(() => import('@/features/reports/components/analytics-chart'), {
  loading: () => <ChartSkeleton />,
})
```

  `ssr: false` only for genuinely browser-only widgets — it removes the component from the server HTML, costing SEO and adding a paint delay. Always give it a `loading` placeholder that holds the layout.
- **Stream with `<Suspense>`** so slow data doesn't block the shell (`data-fetching.md` §3). A dashboard that awaits six queries before sending any HTML is doing the slowest one six times in the user's eyes.
- **Check the bundle before adding a dependency.** A date library or icon set imported wholesale is the usual culprit. Use the analyzer the installed toolchain supports (`@next/bundle-analyzer` for webpack builds; the Turbopack analyzer where available), and `optimizePackageImports` for barrel-heavy packages.

---

## 4. Payload and query discipline

- **Select only what renders.** Precise Prisma `select` on the server; pass only the needed fields into Client Components — props crossing the boundary are serialised into the HTML payload.
- **Paginate every list** (`patterns.md` §6). Cursor pagination for infinite scroll, offset for numbered pages, a hard max on `limit`.
- **N+1:** a `findMany` followed by a per-row lookup. Use `include`/`select` relations, or one `where: { id: { in: ids } }` query.
- **Parallel, not sequential** — independent reads go in `Promise.all` or start early.
- **Cache what's read-mostly** (`'use cache'`) and leave per-user data dynamic.

---

## 5. Accessibility

Non-negotiable for a product people pay for, and cheap while writing the component rather than after.

- **Semantic elements first.** A `<div onClick>` is not a button: no keyboard, no focus, no role. Use `<button>`, `<a>`/`<Link>`, `<nav>`, `<main>`, `<dialog>`.
- **Every interactive element is keyboard-operable** — Tab to reach, Enter/Space to activate, Escape to dismiss.
- **Never remove focus outlines.** Restyle them: `focus-visible:ring-2 focus-visible:ring-offset-2`.
- **Focus management in overlays** — into the dialog on open, trapped while open, back to the trigger on close. shadcn/Radix primitives do this; one more reason not to hand-roll a modal.
- **Label every input** — `<label htmlFor>` or `aria-label`. A placeholder is not a label.
- **Announce async state.** Errors and toasts need `role="alert"`; live regions `aria-live="polite"`. Link field errors with `aria-describedby` and set `aria-invalid`.
- **Route changes:** Next announces page changes to screen readers via the document title — so every page needs a distinct `title`.
- **Contrast:** 4.5:1 body text, 3:1 large text and UI boundaries.
- **Never encode meaning in colour alone.**
- **Respect `prefers-reduced-motion`** (`motion-safe:` / `motion-reduce:` variants).
- Icon-only buttons need an `aria-label`; decorative icons `aria-hidden`.

---

## 6. SEO

Every public page exports metadata:

```ts
// app/(marketing)/pricing/page.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'Pricing',                                  // the layout's template adds "— Acme"
  description: 'Simple per-seat pricing. No setup fees.',
  alternates: { canonical: '/pricing' },
  openGraph: { title: 'Pricing — Acme', description: 'Simple per-seat pricing. No setup fees.', images: ['/og/pricing.png'] },
  twitter: { card: 'summary_large_image' },
}
```

- `metadataBase` set once in the root layout from env, so relative OG URLs resolve — no hardcoded domain.
- `generateMetadata` for dynamic pages; it shares `React.cache`'d data calls with the page, so it doesn't double-query.
- `app/sitemap.ts` and `app/robots.ts` for crawlers; `opengraph-image.tsx` for generated share images.
- `<html lang>` in the root layout. One `<h1>` per page, headings in order.
- Real status codes — `notFound()` for missing pages, not a 200 "not found" screen.
- Pages behind auth need a `title` and nothing more; add `robots: { index: false }` to the authed group's layout.
