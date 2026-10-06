# Analytics & Third-Party Scripts (`@next/third-parties`, `next/script`, GA4)

Read before adding an analytics vendor, a tag manager, or diagnosing numbers in an analytics dashboard that look wrong.

---

## 1. Know the path a hit takes — before anything else

A tracking hit either goes **browser → vendor** or **browser → our server → vendor**. Every analytics diagnosis starts by establishing which, because the second path changes what the vendor sees.

In Next.js the second path appears when someone adds a **proxy for ad-blocker resilience**: a `rewrites()` entry in `next.config` (`/stats/:path*` → `https://region1.google-analytics.com/:path*`), a Route Handler forwarding hits, or a platform-level proxy. `@next/third-parties/google` on its own loads GA directly from Google.

Cheapest runtime check, for the developer: live site → DevTools → Network → filter `collect`. Request host is our own domain → proxied. `region1.google-analytics.com` (or similar) → direct.

---

## 2. Proxied analytics geolocates to the server

The vendor geolocates the IP that **connects** to it. Through a proxy, that is the server or edge node. GA4's browser collection endpoint does not use a forwarded client IP — observed in production: every visitor attributed to one country where the host is.

**Signature:**

| Signal | Proxy geolocation | Ghost spam | Real bot on our pages |
| :--- | :--- | :--- | :--- |
| Country spread | Collapses to 1–2 countries, one implausible | Adds a country on top of a normal spread | Adds a country on top of a normal spread |
| Hostname (GA4 Explore) | Our domain | Not ours, or `(not set)` | Our domain |
| Sources | Real: google, bing, linkedin, chatgpt.com | Junk or `(direct)` | Mostly `(direct)` |
| Engagement | Normal | ~0s | ~0s |
| Pages | Real pages, real locales | Often none / fake | Crawl-shaped |
| vs Search Console geography | Disagrees wholesale | Agrees apart from the spam country | Agrees apart from the bot country |

**Do not diagnose spam or bots from a country chart alone.** Sessions-by-source and hostname separate them in one look.

Dev traffic makes it worse: hits from `next dev` land in the production property. Keep dev out (§3).

---

## 3. Registering a vendor — the canonical setup

```tsx
// app/layout.tsx
import { GoogleAnalytics } from '@next/third-parties/google'

const GA_ID = process.env.NEXT_PUBLIC_GA_ID

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
      {/* production only, so local visits never reach the live property */}
      {process.env.NODE_ENV === 'production' && GA_ID ? <GoogleAnalytics gaId={GA_ID} /> : null}
    </html>
  )
}
```

Every line is a decision:

- **`@next/third-parties`** loads the tag after hydration without blocking the page, and sends page views on client-side navigations. Prefer it over a hand-written `<Script>` with an inline `gtag` snippet. For other vendors, `next/script` with `strategy="afterInteractive"` (or `lazyOnload` for non-critical widgets).
- **Production only.** Test a real hit with `next build && next start` and the env set.
- **Direct, not proxied** — unless the vendor documents honouring a forwarded client IP (§2). For GA4 the supported first-party route is server-side GTM, not a rewrite.
- **`NEXT_PUBLIC_GA_ID` is inlined at build time.** It must be set in the **build** environment; setting it only in the runtime host does nothing until the next build (`smells.md` §3). A missing ID means no tag — the guard above makes that a visible absence rather than a crash, so check the deployed HTML for the `gtag/js?id=G-…` script after the first deploy.
- **No `|| ''` and no hardcoded ID.** A fallback hides a missing production ID; a literal ID in source ships the production property into every preview deploy.

**Before deploying:**

- **Env:** the real `G-…` ID is set in the build environment for production builds only (not previews, unless intended).
- **CSP:** direct hits need `*.google-analytics.com` and `*.googletagmanager.com` in `connect-src` / `script-src`. A CSP block fails silently.
- **Consent:** if the site needs consent mode, wire it before the tag loads — never "add it later".

**After deploying:** Network → filter `collect` shows requests to the vendor host carrying the `G-…` ID, and GA4 Realtime shows the visit within a minute with a plausible country.

**Vercel Analytics / Speed Insights** and other first-party products measure different things (and count differently) — don't compare their numbers to GA without naming the difference (§5).

---

## 4. What is damaged, and what is not

Tell the developer this in plain terms — they will be asked by someone non-technical.

- **GA4 never reprocesses.** Data collected through a proxy keeps the wrong country forever. The fix is forward-only; annotate the fix date.
- **Wrong:** country / region / city reports, key events by geography, location-based audiences, Ads geo-targeting if linked.
- **Still usable:** user and session counts, sources, pages, engagement.
- **Leads are fine** when forms post to our own Server Action (`email-pdf.md`) — GA is a reporter, not the system of record.

---

## 5. Diagnosing any analytics discrepancy

Same Diagnose-mode discipline as any bug (`SKILL.md`). Two extra rules:

1. **Dashboards are readers.** A CMS dashboard plugin runs a plain GA4 Data API query and shows what the property holds. Don't debug the plugin; check the query has no filters and move on to the property.
2. **Sources that count different things never match.** Search Console = Google Search clicks, 28-day default. GA4 = all users, the report's window, only since the tag went live. Name the mismatch in units and window before calling anything missing.

The opening pair is almost always: *"Either the traffic is fake (spam / bot), or our pipeline is rewriting real traffic (proxy, dev hits, consent mode, a filter)."* The cheapest separator is the developer reading Sessions by source and Hostname for the suspect segment.

**No `collect` request at all** on the live site: the tag never ran. Check, in order: the ID was present at **build** time (view source for the script), CSP, consent blocking it, and the Console for an error that stopped hydration.

---

## 6. Strapi GA dashboard plugin — setup

For `strapi-google-analytics-dashboard` or similar. The plugin passes credentials straight to Google's `BetaAnalyticsDataClient`. Three values are easy to mix up:

| Field | Value | Where from |
| :--- | :--- | :--- |
| Property ID | A **number**, not `G-…` | GA4 → Admin → Property details, or the `p123456789` in the GA URL |
| Measurement ID | `G-…` | The web data stream |
| Credentials | The **whole** service-account JSON key, pasted unedited (`\n` inside `private_key` stays as is) | Google Cloud → IAM → Service accounts → Keys → JSON |

Prerequisites, in order:
1. Enable the **Google Analytics Data API** in the Cloud project.
2. Create the service account. It needs no Cloud roles.
3. Add its email in **GA4** → Property access management as **Viewer**. Skipping this is the usual cause of "invalid credentials".
4. Rebuild the Strapi admin after installing (`npm run build`).

Reading the result:

- **"Invalid credentials or property ID"** — Viewer access missing or not yet propagated, or `G-…` pasted as the Property ID.
- **"No data"** — credentials **accepted**. Either the site isn't sending hits yet (§3), or GA's standard reports haven't caught up (24–48 h). Realtime confirms hits long before the plugin does.

**The key file is a password.** Never committed, never pasted into chat, deleted locally once saved in Strapi. Restrict Strapi Settings to Super Admins.
