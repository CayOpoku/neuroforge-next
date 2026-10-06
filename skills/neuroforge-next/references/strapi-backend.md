# Strapi Backend: Config, Preview & Generated Types

Read before touching a Strapi project — first boot, a new content type, a preview bug, or any CSP/CORS failure between Strapi and Next.js. Assumes a **two-app setup**: a Strapi app (its own folder, its own `package.json`, its own `.env`) and a Next.js app that consumes it.

---

## 1. Ownership gate — Strapi replaces Prisma, it does not sit beside it

When Strapi is the backend, **Strapi owns the database and the content schema.** Consequences, all non-negotiable:

- **No Prisma.** Do not add `prisma/schema.prisma` next to a Strapi app to "also" model content. Two ORMs on one database is a split brain, not a layer. `patterns.md` §3 (Prisma singleton) does not apply to Strapi projects.
- **Next.js server code is a BFF, not a data layer.** Server Components, Server Actions and Route Handlers hide the Strapi token, run the draft-mode handshake, and reshape payloads. They never open a database connection.
- **Content modelling happens in Strapi**, either through the admin UI or by hand-editing `src/api/**/content-types/**/schema.json`. Either way §7 applies.

A schema change in Strapi is a **Tier 2** change — it moves the data contract for both apps.

---

## 2. What must be configured on a fresh Strapi app

Four files, in this order. `create-strapi-app` gives you none of the frontend integration; every one of these is a step the generator skips.

| File | Why it cannot be left at default |
| :--- | :--- |
| `config/middlewares.ts` | Default CSP blocks the Next.js preview iframe and every external image host. |
| `config/admin.ts` | Preview is off by default; secrets must come from env, never literals. |
| `config/plugins.ts` | Local plugins are not auto-discovered — they need an explicit `resolve`. |
| `.env` | `CLIENT_URL` and `PREVIEW_SECRET` are consumed by the two files above. |

All config files are typed. Use `Core.Config.*` and let the compiler catch a misplaced key — an untyped `export default ({ env }) => ({ ... })` silently accepts a typo and Strapi boots with the block ignored.

---

## 3. `config/middlewares.ts` — CSP is where the integration silently fails

The default `strapi::security` policy refuses to be framed and refuses foreign image hosts. Symptom: the preview panel renders a blank iframe with a console CSP violation, and remote images or map tiles never appear in the admin.

```ts
import type { Core } from '@strapi/strapi'

const config = ({ env }: Core.Config.Shared.ConfigParams): Core.Config.Middlewares => [
  'strapi::errors',
  {
    name: 'strapi::security',
    config: {
      contentSecurityPolicy: {
        useDefaults: true,
        directives: {
          'img-src': [
            "'self'",
            'data:',
            'blob:',
            '*.tile.openstreetmap.org',
            'tile.openstreetmap.org',
          ],
          'frame-ancestors': ["'self'", env('CLIENT_URL')],
          'frame-src': ["'self'", env('CLIENT_URL')],
          'connect-src': ["'self'", 'https:', 'http:', env('CLIENT_URL')],
          upgradeInsecureRequests: null,
        },
      },
    },
  },
  'strapi::cors',
  'strapi::poweredBy',
  'strapi::logger',
  'strapi::query',
  'strapi::body',
  'strapi::session',
  'strapi::favicon',
  'strapi::public',
]

export default config
```

Rules for editing this block:

- **Order is load order.** `strapi::errors` first, `strapi::public` last. Replacing the array with only the entries you care about disables the rest — always expand the full default list and swap the single entry.
- **`frame-ancestors` + `frame-src` are the preview pair.** `frame-ancestors` lets the Next.js page be framed by admin; `frame-src` lets admin frame it. Missing either gives a blank preview panel. Note the quoting: CSP keywords are quoted inside the string (`"'self'"`), origins are not.
- **`upgradeInsecureRequests: null`** disables the directive so an `http://localhost:3001` frontend stays reachable in development. Left on, the browser rewrites the preview URL to `https://` and the iframe dies. Both apps are HTTPS in production, so this is a dev concession — flag it, don't celebrate it.
- **`img-src` is an allowlist, not a wildcard.** Add the hosts the project actually uses (tile server, Cloudinary, S3 bucket). `'*'` is a finding.
- **`connect-src` with `'https:'` and `'http:'` is broad.** Tolerable while the set of API hosts is still moving; narrow it before production and say so out loud.
- **CSP is not CORS.** `strapi::cors` governs whether browser-side requests from the Next.js app may call Strapi (server-side fetches aren't subject to CORS); CSP governs what the *admin browser page* may load and frame. A working fetch plus a blank preview iframe is the normal way to discover you fixed only one of the two.

---

## 4. `config/admin.ts` — secrets from env, and the preview handler

```ts
import type { Core } from '@strapi/strapi'

const config = ({ env }: Core.Config.Shared.ConfigParams): Core.Config.Admin => ({
  auth: { secret: env('ADMIN_JWT_SECRET') },
  apiToken: { salt: env('API_TOKEN_SALT') },
  transfer: { token: { salt: env('TRANSFER_TOKEN_SALT') } },
  secrets: { encryptionKey: env('ENCRYPTION_KEY') },
  flags: {
    nps: env.bool('FLAG_NPS', true),
    promoteEE: env.bool('FLAG_PROMOTE_EE', true),
  },
  preview: {
    enabled: true,
    config: {
      allowedOrigins: [env('CLIENT_URL')],
      async handler(uid, { documentId, locale, status }) {
        const baseUrl = env('CLIENT_URL')
        const secret = env('PREVIEW_SECRET')

        const document = (await strapi.documents(uid as any).findOne({
          documentId,
          locale,
        })) as { slug?: string } | null

        if (!document) return null

        // Strapi UID -> Next.js route. One map, mirroring the routes it points at.
        const basePaths: Record<string, string> = {
          'api::page.page': '/',
          'api::article.article': '/blog',
          'api::career.career': '/careers',
          'api::privacy-policy.privacy-policy': '/privacy-policy',
        }

        const basePath = basePaths[uid] ?? '/'
        const slug = document.slug ?? ''

        let pathname = basePath === '/' ? `/${slug}` : `${basePath}/${slug}`
        if (slug === 'homepage' || slug === 'home') pathname = '/'
        pathname = pathname.replace(/\/+/g, '/')

        return `${baseUrl}/api/draft?slug=${encodeURIComponent(pathname)}&status=${status}&secret=${secret}&locale=${locale}`
      },
    },
  },
})

export default config
```

What matters here:

- **Every secret is `env(...)`.** A literal in `admin.ts` is a committed secret — critical finding, same rule as `smells.md` §3. `ADMIN_JWT_SECRET`, `API_TOKEN_SALT`, `TRANSFER_TOKEN_SALT` and `ENCRYPTION_KEY` have no safe default, and rotating them invalidates existing admin sessions and API tokens — never regenerate them on a running project to "fix" a boot error.
- **`allowedOrigins` is a second gate on top of CSP.** The preview URL's origin must appear here *and* in `frame-ancestors`, or admin refuses to open it.
- **The handler returns a URL string or `null`.** `null` means no preview button for that document. Returning a wrong URL instead of `null` produces a preview panel that 404s — a worse failure than no button.
- **The handler runs on the Strapi server**, so the `strapi` global is available. It fetches the document because the handler only receives `documentId`; the slug it needs for the URL is not passed in.
- **It points at a Next.js Route Handler, never a page directly.** `/api/draft` performs the handshake — verify `secret`, enable Draft Mode, redirect to `slug` — and only then does the real page render draft content. Linking straight to `/blog/my-post` shows the published version inside the iframe and produces a confusing bug report.
- **`status`** is `'draft' | 'published'`; forward it, the Next.js side needs it to choose the Strapi query status.
- **`uid as any`** is the one tolerated cast in this file: `strapi.documents()` is typed against the generated UID union while `uid` arrives widened. Do not let the `any` spread — the result is immediately narrowed with `as { slug?: string } | null`.
- **The `basePaths` map is duplicated knowledge.** It mirrors the Next.js route tree, and it is the thing everyone forgets when a route moves. Any new content type with a public page gets its entry in the same change.

**Single types** (`about-page`, `privacy-policy`) usually have no `slug`. Give them a fixed path and skip the concatenation rather than shipping a URL ending in `/undefined`.

---

## 5. `config/plugins.ts` — local plugins need an explicit resolve

```ts
import type { Core } from '@strapi/strapi'

const config = ({ env }: Core.Config.Shared.ConfigParams): Core.Config.Plugin => ({
  'import-export': {
    enabled: true,
    resolve: './src/plugins/import-export',
  },
})

export default config
```

- **`resolve` is only for local plugins** living in `src/plugins/`. An npm plugin is discovered by package name and takes `enabled` (plus `config`) only — adding `resolve` to it breaks the boot.
- **The key is the plugin's own name**, which is not always the folder name. Check the plugin's `strapi-server` registration when they differ.
- Configure npm plugins here too — upload provider, i18n, SEO. One file, no second config location.

---

## 6. `.env` — the two keys the integration depends on

```bash
CLIENT_URL=http://localhost:3001
PREVIEW_SECRET=replace-with-a-long-random-string
```

- **`.env` is a protected file.** Never write to it unprompted (SKILL non-negotiable #2). Print the keys the user must add, and update `.env.example` in the same breath — that file is not protected and is the only place these key names are discoverable by the next developer.
- **`CLIENT_URL` has no default and must not be given one.** It is read by CSP, `allowedOrigins` and the preview handler; a `|| 'http://localhost:3000'` fallback turns a missing production value into a preview pointed at nothing — the anti-pattern in `smells.md` §3.
- **No trailing slash.** It is concatenated with an already-leading-slash pathname, and `//blog/x` also fails an exact origin comparison.
- **`PREVIEW_SECRET` must be identical on both sides.** Strapi puts it in the URL; the Next.js Route Handler compares it and 401s on mismatch. It travels in a query string, so treat it as a handshake token, not a credential: rotate it freely, and never reuse an API token or JWT secret for it.
- Strapi's own boot secrets live here too (`APP_KEYS`, `ADMIN_JWT_SECRET`, `API_TOKEN_SALT`, `TRANSFER_TOKEN_SALT`, `ENCRYPTION_KEY`, `DATABASE_*`). All are per-environment; none are shared with the Next.js app.

---

## 7. Generated types — `contentTypes.d.ts` and `components.d.ts`

`types/generated/contentTypes.d.ts` and `types/generated/components.d.ts` are Strapi's compiled view of every schema. **They are the contract the Next.js app types itself against**, so a stale file means the frontend is typed against a schema that no longer exists — and TypeScript will confidently confirm the wrong shape.

**They are generated. Never hand-edit them.** An edit survives until the next boot, then vanishes along with the bug it was hiding.

### When to check them

Any time a schema changes — and schemas change in more ways than people expect:

- a content type created, renamed or deleted in the admin UI
- a field added, removed or retyped in the admin UI
- **a hand-edited `src/api/**/content-types/**/schema.json`** — the case that bites, because nothing in the UI signals it
- a component created or moved between categories (`src/components/<category>/<name>.json`)
- a relation changed on *either* side — both content types' entries move
- i18n or draft & publish toggled on a type

### The loop

```bash
npm run develop        # boot regenerates types/generated/*.d.ts
git status types/      # did they actually move?
git diff types/generated/contentTypes.d.ts
```

Then, before writing any Next.js code against the change:

1. **Confirm the UID exists** in `contentTypes.d.ts` — `'api::career.career'` — with the exact singular/plural spelling Strapi chose. Guessing the UID is the most common preview and query bug, and it fails at runtime, not at typecheck.
2. **Confirm the field is there and is the type you expect** — optional vs `Schema.Attribute.Required`, relation arity (`OneToMany` vs `ManyToOne`).
3. **Confirm the component UID** in `components.d.ts` — `'shared.seo'` — before using it as a dynamic-zone discriminator on the Next.js side.
4. **Commit the regenerated files with the schema change.** Schema and generated types are one commit; splitting them breaks the next person's typecheck for reasons their diff does not explain.

If the files do not regenerate, the app did not actually boot — read the error, a malformed hand-written `schema.json` fails in the middle of noisy output — or TypeScript is not enabled for the project. Do not work around it by writing the interface by hand.

The Next.js app usually lives in a separate repo or workspace package: copy or publish the generated `.d.ts` files to it as part of the schema change (a script in `scripts/`, or a shared `packages/cms-types`) — a frontend typed against last month's copy is the same bug.

### Deriving Next.js types from them

Import from the generated types rather than re-declaring the shape in `types/`:

```ts
type Career = ApiCareerCareer['attributes']
```

Hand-written mirrors of Strapi entities drift within a sprint, and they are the single largest source of duplicated types in a Strapi/Next.js repo. Per `type-safety.md`, a type that can be derived is never re-declared. Where the generated types are awkward to consume directly, write **one** mapping layer next to the fetch and derive everything downstream from it.

---

## 8. New-project checklist

1. `config/middlewares.ts` — `frame-ancestors`, `frame-src`, `img-src` hosts, `upgradeInsecureRequests: null` for dev.
2. `config/admin.ts` — secrets via `env()`, `preview.enabled`, `allowedOrigins`, handler with the UID→path map.
3. `config/plugins.ts` — local plugins with `resolve`, npm plugins with `enabled`.
4. `.env` + `.env.example` — `CLIENT_URL`, `PREVIEW_SECRET` (ask before writing `.env`).
5. `config/server.ts` / `config/database.ts` — confirm `HOST`/`PORT` and the DB client match the deployment target before the first migration.
6. A **read-only** API token created in admin for published reads, stored server-side in Next.js only (never `NEXT_PUBLIC_`). The preview path uses `PREVIEW_SECRET`, not this token.
7. Boot once, verify `types/generated/*.d.ts` exist, commit them.
8. Verify preview end to end **from the admin panel** — not by opening the preview URL in its own tab, which bypasses `frame-ancestors` and hides the exact failure this file exists to prevent.

---

## 9. Smells to flag in a Strapi audit

1. **Secrets as literals** in `admin.ts`, `database.ts` or `plugins.ts`.
2. **`middlewares.ts` reduced to a partial array** — silently disables `strapi::errors`, `strapi::body` or `strapi::public`.
3. **`'*'` in `img-src`, `frame-ancestors` or `allowedOrigins`** — a wildcard framing policy is a clickjacking finding, not a convenience.
4. **A Strapi API token reachable from the client** — in a `NEXT_PUBLIC_` variable, a Client Component, or any fetch that runs in the browser. Tokens live in server-only modules.
5. **Hand-written interfaces mirroring content types** while `contentTypes.d.ts` sits generated and unused.
6. **Generated types uncommitted or out of date** against `src/api/**/schema.json`.
7. **A `basePaths` entry missing** for a content type that has a public page — preview silently falls back to `/`.
8. **A preview URL built without the secret**, or a Next.js draft route that accepts the request without comparing it.
9. **Prisma or a raw DB client** in a repo where Strapi owns the schema (§1).
10. **Business logic in the frontend** that belongs in a Strapi controller or service override — filtering, permission checks or aggregation done in Next.js over an over-fetched list.
