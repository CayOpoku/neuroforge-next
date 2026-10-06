# Next.js ⇄ Strapi: Pages, Dynamic Zones & Draft Mode

The frontend half of `strapi-backend.md`. Read before building a page against Strapi, adding a block, wiring preview, or debugging a blank/404 page. Assumes `data-fetching.md` for the fetching and caching rules this file builds on.

---

## 1. Mode gate — resolve this before creating a single file

Two architectures. They are not a progression, and mixing them is how a marketing site ends up with three ways to render a hero.

| | **Mode S — single types** | **Mode C — collection + dynamic zone** |
| :--- | :--- | :--- |
| Strapi shape | One single type per page (`homepage`, `about-page`, `contact`) | One `pages` collection, each entry a `dynamic_zone` of components |
| Next.js shape | One `page.tsx` per page, fixed layout | `app/[[...slug]]/page.tsx` resolving blocks at runtime |
| Editors can | Edit copy and images in a fixed layout | Compose new pages from blocks without a deploy |
| Cost | Near zero architecture | Registry, block boundaries, populate strategy, lint rules |
| Correct when | The page set is small, fixed, and designed per-page | Editors add pages, or layouts repeat across pages |

**Mode S is the default.** A six-page brochure site of single types needs no registry and no block configs — a page component and one typed fetch each.

**Escalate to Mode C when the answer to "can an editor publish a new page without a developer?" must be yes**, or when the same section appears on several pages with different content.

Record the mode in `neuroforge/00-project-overview.md` alongside Offline Mode. Per-route exceptions are normal: `app/blog/[slug]/page.tsx` rendering an `article` collection with a fixed layout is Mode S inside a Mode C project.

---

## 2. The Strapi client — server-only, one place

```ts
// server/cms.ts
import 'server-only'
import qs from 'qs'
import { requiredEnv } from '@/server/env'
import { ApiError } from '@/lib/api'

const CMS_URL = requiredEnv('CMS_URL')                 // no trailing slash, no fallback
const CMS_TOKEN = requiredEnv('CMS_READ_TOKEN')        // read-only; draft scope only if drafts are read here

export type StrapiStatus = 'draft' | 'published'

export async function cmsGet<T>(path: string, query: Record<string, unknown> = {}): Promise<T> {
  const search = qs.stringify(query, { encodeValuesOnly: true })
  const res = await fetch(`${CMS_URL}/api/${path}${search ? `?${search}` : ''}`, {
    headers: { Authorization: `Bearer ${CMS_TOKEN}` },
  })
  const body: unknown = await res.json().catch(() => null)
  if (!res.ok) throw new ApiError(res.status, body)     // getErrorMessage reads Strapi's { error: { message } }
  return body as T
}
```

- **The token never reaches the browser** — not `NEXT_PUBLIC_`, not a Client Component, not a client fetch. Every CMS read happens in Server Components or server modules.
- One client, one place. `@strapi/client` is a fine alternative if the project uses it — wrap it the same way.
- Type responses from the generated Strapi types (`strapi-backend.md` §7), never a hand-written mirror.

### Caching CMS reads

```ts
// features/cms/server/pages.ts
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'

export async function getPageBySlug(slug: string, locale: string, status: StrapiStatus) {
  'use cache'
  cacheLife(status === 'draft' ? 'seconds' : 'hours')
  cacheTag('cms', `cms:page:${slug}`)
  return cmsGet<StrapiPageResponse>('pages', {
    filters: { slug: { $eq: slug } },
    status,
    locale,
    populate: PAGE_POPULATE,
  })
}
```

- **`status` and `locale` are arguments**, so draft and published are separate cache entries — a cached draft must never be served to the public.
- **Invalidate from a Strapi webhook** — a Route Handler (`app/api/revalidate/route.ts`) verifying a shared secret and calling `revalidateTag('cms', 'max')` (or the specific page tag). Without it, edits appear only when `cacheLife` expires, and editors report "publishing is broken".
- On projects without `cacheComponents`, use the project's model (`fetch` with `next: { tags }`).

---

## 3. Mode S — single-type pages, kept boring

```tsx
// app/(marketing)/about/page.tsx
import type { Metadata } from 'next'
import { getAboutPage } from '@/features/cms/server/about'

export async function generateMetadata(): Promise<Metadata> {
  const page = await getAboutPage()
  return toMetadata(page.data.seo, '/about')          // §9
}

export default async function AboutPage() {
  const page = await getAboutPage()
  return <AboutView page={page.data} />
}
```

- The populate map lives **next to the fetch that needs it**, not in a shared `POPULATION_MAPS` object.
- `generateMetadata` and the page call the same function — wrap it in `React.cache` (or rely on `'use cache'`) so it runs once per request.
- No registry, no `__component` switch. Sections are ordinary components with typed props.

Everything below is Mode C.

---

## 4. Mode C — the block layout

**One block = one folder.** Adding a block touches its folder and one line in the registry.

```
features/cms/
  blocks/
    hero/typing-collage/
      index.tsx          # Server Component — layout only
      config.ts          # Strapi UID + populate
      collage.client.tsx # only if the block has interactivity
    content/feature-grid/
      index.tsx
      config.ts
  registry.ts            # UID → component, and the derived populate map
  server/pages.ts        # the cached fetch (§2)
  types/blocks.types.ts  # the discriminated union (§6)
app/
  [[...slug]]/page.tsx   # orchestrator only (§5)
```

### Import boundaries

| From | May import |
| :--- | :--- |
| `blocks/<cat>/<name>/index.tsx` | `components/`, `lib/`, `features/cms/types/`, its own folder |
| `blocks/<cat>/<name>/config.ts` | `features/cms/types/` — **nothing else** |
| `components/ui/*` | external packages only |

**A block never imports another block.** Shared markup moves to `components/app/` or `components/content/`. Enforce with `eslint-plugin-boundaries` — a boundary nobody checks is already broken.

Blocks are **Server Components by default** — they ship no JS. Interactivity goes in a small `*.client.tsx` inside the block folder, so a page only ships JS for the interactive blocks it renders.

Category folders mirror Strapi's component categories (`hero.typing-collage` → `blocks/hero/typing-collage/`), so the UID in an error message tells you the folder.

---

## 5. The registry — one list, two maps

There's no build-time glob in the Next.js toolchain to rely on, so the registry is an **explicit list** — and it must also produce the populate map, because a hand-maintained populate object is the file everyone forgets. A missing populate entry is invisible: the block renders with empty fields, in production.

```ts
// features/cms/types/block-config.types.ts
export interface BlockConfig {
  /** Strapi component UID, exactly as it appears in `__component` */
  uid: string
  /** Populate tree for this block's own relations. Omit when it has none. */
  populate?: Record<string, unknown>
}
```

```ts
// features/cms/blocks/hero/typing-collage/config.ts
import type { BlockConfig } from '@/features/cms/types/block-config.types'

export const config = {
  uid: 'hero.typing-collage',
  populate: { images: { populate: '*' }, cta: true },
} satisfies BlockConfig
```

```tsx
// features/cms/registry.ts
import type { ComponentType } from 'react'
import type { StrapiBlock } from '@/features/cms/types/blocks.types'
import * as HeroTypingCollage from './blocks/hero/typing-collage'
import * as ContentFeatureGrid from './blocks/content/feature-grid'

// Add a block: create its folder, then one line here.
const entries = [HeroTypingCollage, ContentFeatureGrid] as const

export const blockComponents: Record<string, ComponentType<{ data: StrapiBlock }>> = Object.fromEntries(
  entries.map((m) => [m.config.uid, m.default as ComponentType<{ data: StrapiBlock }>]),
)

/** Strapi v5 dynamic-zone population: one `on` entry per registered block. */
export const dynamicZonePopulate = {
  on: Object.fromEntries(entries.map((m) => [m.config.uid, m.config.populate ? { populate: m.config.populate } : true])),
}

export const PAGE_POPULATE = { dynamic_zone: dynamicZonePopulate, seo: { populate: '*' } } as const
```

Each block folder's `index.tsx` default-exports the component and re-exports `config` (`export { config } from './config'`).

- **The UID lives in the config, not in a folder-name convention.**
- **Verify each UID against `components.d.ts`** (`strapi-backend.md` §7). A typo here is not a type error — add a unit test that every registry UID exists in the generated component types, so CI catches it.

---

## 6. `app/[[...slug]]/page.tsx` — an orchestrator, and nothing else

```tsx
import { notFound } from 'next/navigation'
import { draftMode } from 'next/headers'
import { blockComponents } from '@/features/cms/registry'
import { getPageBySlug } from '@/features/cms/server/pages'

type Props = { params: Promise<{ slug?: string[] }> }

export default async function CmsPage({ params }: Props) {
  const { slug: parts } = await params
  const slug = parts?.join('/') || 'homepage'
  const { isEnabled } = await draftMode()

  const res = await getPageBySlug(slug, 'en', isEnabled ? 'draft' : 'published')
  const page = res.data[0]
  if (!page) notFound()                                  // a real 404, decided before anything streams

  return (
    <>
      {page.dynamic_zone.map((block) => {
        const Block = blockComponents[block.__component]
        if (Block) return <Block key={`${block.__component}-${block.id}`} data={block} />
        if (process.env.NODE_ENV === 'development') return <BlockMissing key={block.id} uid={block.__component} />
        return null                                      // deploy behind the CMS: render nothing, monitor it
      })}
    </>
  )
}
```

- **`notFound()`, never an in-page "not found" component** — that's a soft 404: the crawler gets 200 and indexes an error screen. Brand the 404 in `not-found.tsx`.
- **CMS unreachable is thrown**, landing in `error.tsx` — not rendered as an empty page.
- **Unknown `__component`** in production renders nothing — a missing section beats a broken-looking page. Log it server-side so monitoring catches it.
- `generateStaticParams` can prebuild published slugs; new pages still render on demand.
- **Never write error payloads into a cookie** or global state — that ships internal detail to the browser on every request.

---

## 7. Types — one discriminated union, zero `any`

```ts
// features/cms/types/blocks.types.ts
export interface StrapiImage { url: string; alternativeText?: string | null; width?: number; height?: number }

interface BlockBase { id: number; __component: string }

export interface HeroTypingCollage extends BlockBase {
  __component: 'hero.typing-collage'
  headline: string
  images?: StrapiImage[]
}

export interface ContentFeatureGrid extends BlockBase {
  __component: 'content.feature-grid'
  title?: string
  features?: { id: number; label: string }[]
}

/** Every block the app can render. Add the interface here when you add the folder. */
export type StrapiBlock = HeroTypingCollage | ContentFeatureGrid
```

Each block narrows its own prop:

```tsx
export default function HeroTypingCollageBlock({ data }: { data: StrapiBlock }) {
  if (data.__component !== 'hero.typing-collage') return null
  // data is HeroTypingCollage from here
}
```

- **`type CMSData = any` is an opt-out**, and one `any` at the block boundary propagates into every component that touches `data`. If a payload can't be described yet, `unknown` plus a narrowing function (`type-safety.md` §1).
- **`seo?: StrapiSeo | StrapiSeo[]`** — normalise once in the fetch layer so the rest of the app sees a single object.
- Prefer deriving these from the generated Strapi types where they're consumable; write one mapping layer next to the fetch rather than a mirror per component.

---

## 8. Draft Mode — the handshake route

This is the route `config/admin.ts` points at (`strapi-backend.md` §4). It is the only place the secret is compared.

```ts
// app/api/draft/route.ts
import { draftMode } from 'next/headers'
import { redirect } from 'next/navigation'
import { z } from 'zod'

const QuerySchema = z.object({
  slug: z.string().startsWith('/'),                       // stops ?slug=https://evil.example — an open redirect
  secret: z.string(),
  status: z.enum(['draft', 'published']).default('draft'),
  locale: z.string().optional(),
})

export async function GET(request: Request) {
  const parsed = QuerySchema.safeParse(Object.fromEntries(new URL(request.url).searchParams))
  if (!parsed.success) return Response.json({ message: 'Invalid preview request' }, { status: 400 })

  const { slug, secret, status, locale } = parsed.data
  const expected = process.env.PREVIEW_SECRET
  if (!expected || secret !== expected) return Response.json({ message: 'Invalid preview secret' }, { status: 401 })

  const draft = await draftMode()
  if (status === 'draft') draft.enable()
  else draft.disable()

  redirect(locale ? `/${locale}${slug}` : slug)
}
```

- **The secret is server env (`PREVIEW_SECRET`), never `NEXT_PUBLIC_`**, and never compared in a Client Component — a public secret lets anyone read unpublished content.
- **Pages ask `draftMode().isEnabled`**, nothing else. The client never decides the status.
- **The iframe needs a cross-site cookie.** Strapi admin is a different origin, so the draft cookie must be `SameSite=None; Secure` to be sent on the framed request. Check the `__prerender_bypass` cookie's attributes in DevTools on the deployed environment; over plain `http://localhost` browsers reject `Secure` cookies, so verify preview against an HTTPS environment (`strapi-backend.md` §3 has the matching CSP concession).
- **Pair it with an exit route** (`app/api/draft/disable/route.ts` → `(await draftMode()).disable()` + redirect) and a visible "Exit preview" banner when `isEnabled`, or editors keep seeing drafts on the public site.
- **Draft reads need a token with draft permission** — keep it server-only and use it only when `isEnabled`; published reads use the read-only token.

---

## 9. Global settings and SEO

A cached `getGlobal(locale)` server function fetches the site-wide single type (name, favicon, `defaultSeo`). Pages call it freely — `'use cache'`/`React.cache` makes repeated calls free.

```ts
// features/cms/server/seo.ts
import type { Metadata } from 'next'

export async function toMetadata(seo: StrapiSeo | null, path: string): Promise<Metadata> {
  const { defaultSeo } = await getGlobal('en')
  const title = seo?.metaTitle ?? defaultSeo?.metaTitle
  const image = absoluteMediaUrl(seo?.metaImage?.url ?? defaultSeo?.metaImage?.url)
  return {
    title,
    description: seo?.metaDescription ?? defaultSeo?.metaDescription,
    alternates: { canonical: path },
    openGraph: { title, url: path, type: 'website', images: image ? [image] : undefined },
    twitter: { card: 'summary_large_image' },
  }
}
```

- `metadataBase` in the root layout makes relative URLs absolute — **no hardcoded domain** in a fallback chain.
- **`absoluteMediaUrl` is one util** in `lib/`: return absolute URLs unchanged, otherwise prefix the CMS media origin. Every OG image, `next/image` source and download link uses it — and the CMS media host goes in `images.remotePatterns`.

---

## 10. Smells to flag on the Next.js side

1. **Preview secret or a draft-scoped token in `NEXT_PUBLIC_*`**, or a client-side secret comparison — critical, it exposes unpublished content.
2. **Any CMS fetch from a Client Component** carrying a token.
3. **Draft and published sharing a cache entry** — `status` not part of the cached function's arguments.
4. **No revalidation webhook** — edits appear only when the cache expires.
5. **A soft 404** — an in-page not-found component instead of `notFound()`.
6. **A hand-maintained populate map** alongside a registry that could derive it (§5).
7. **`any` / `CMSData` at the block boundary**, or an `eslint-disable` in a shared types file.
8. **A block importing another block**, or a config importing a component.
9. **Client-component blocks by default** — `'use client'` on a whole block that only needs one interactive child.
10. **Hardcoded domains, site names or CMS URLs** in components.
11. **`populate: '*'` on a page query** — one level deep, silently misses nested block relations and over-fetches what it reaches.
12. **A registry UID with no matching Strapi component** (or vice versa) — verify against `components.d.ts`, ideally in a test.
