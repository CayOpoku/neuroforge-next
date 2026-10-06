# Email & File Delivery (Nodemailer, React Email, Resend, Strapi)

Read before sending a mail, wiring a contact form, building a newsletter, or serving a PDF. Applies to Next.js-only projects and to Strapi-backed ones.

---

## 1. Ownership gate — one app owns sending

Strapi ships an email plugin backed by Nodemailer. If it is already configured, **do not add a second set of SMTP credentials to Next.js.** Two senders means two credential rotations, two suppression lists, two places to look when a mail does not arrive, and two `From` identities landing in spam for different reasons.

| Situation | Sender |
| :--- | :--- |
| Strapi is present and already sends (admin invites, password resets) | **Strapi.** Next.js calls it server-to-server. |
| The email body is content editors own (templates, campaigns, localised copy) | **Strapi** — the template lives with the content. |
| No Strapi, or the mail is app-owned (auth flows, receipts, a form that stores nothing) | **Next.js server code** (§3). |
| Bulk / marketing / anything with a list | **Neither directly** — an ESP (§5). |

Whichever you pick, **credentials live in one app's server env only.** SMTP or ESP keys in `NEXT_PUBLIC_*`, a Client Component, or any client-reachable payload are a critical finding.

---

## 2. Strapi sends, Next.js calls it

Configure the provider once in Strapi's `config/plugins.ts` (`strapi-backend.md` §5), expose a narrow custom route there (a controller that validates, stores and sends), and call it from a **Server Action**, never from the browser:

```ts
// features/contact/actions/submit-contact.ts
'use server'
import { cmsPost } from '@/server/cms'

export async function submitContactAction(_prev: ActionResult | null, formData: FormData): Promise<ActionResult> {
  const parsed = ContactSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return validationFailure(parsed.error)
  if (parsed.data.website) return { ok: true, data: undefined }  // honeypot — log it (§3 "Honeypot")

  await cmsPost('contact-submissions', { data: parsed.data })   // token attached server-side in server/cms.ts
  return { ok: true, data: undefined }
}
```

Strapi stores the submission and sends the mail in one lifecycle hook — the record and the notification cannot drift apart. Pick one owner per side effect.

---

## 3. Next.js sends

### Templates are React Email components

```tsx
// templates/email/contact-notification.tsx
import { Html, Body, Container, Heading, Text } from '@react-email/components'

export function ContactNotification({ name, email, message }: { name: string; email: string; message: string }) {
  return (
    <Html>
      <Body>
        <Container>
          <Heading>New contact from {name}</Heading>
          <Text>{email}</Text>
          <Text>{message}</Text>
        </Container>
      </Body>
    </Html>
  )
}
```

React escapes every interpolated value, which removes the HTML-injection bug of template-literal mail bodies. Never build an HTML body with a template string.

### Transport: an ESP SDK or a Nodemailer singleton

With an ESP (Resend, Postmark, Brevo), use its SDK from a server module — Resend accepts a React element directly (`react: <ContactNotification … />`). With SMTP, a **singleton transporter** — one created per request opens a new SMTP connection each time, exhausts the provider's limit under load, and HMR multiplies it in dev:

```ts
// server/mailer.ts
import 'server-only'
import nodemailer, { type Transporter } from 'nodemailer'
import { render, toPlainText } from 'react-email'   // older setups: render from '@react-email/render'
import type { ReactElement } from 'react'

const globalForMailer = globalThis as unknown as { mailer?: Transporter }

function getTransporter(): Transporter {
  if (globalForMailer.mailer) return globalForMailer.mailer

  const { SMTP_HOST, SMTP_PORT, SMTP_USER, SMTP_PASS } = process.env
  if (!SMTP_HOST || !SMTP_USER || !SMTP_PASS) throw new Error('[CONFIG] Mail transport is not configured')

  const port = Number(SMTP_PORT) || 587
  globalForMailer.mailer = nodemailer.createTransport({
    host: SMTP_HOST,
    port,
    secure: port === 465,          // 465 = implicit TLS; 587 = STARTTLS
    auth: { user: SMTP_USER, pass: SMTP_PASS },
    pool: true,
    maxConnections: 3,
  })
  return globalForMailer.mailer
}

export async function sendMail(opts: { to: string; subject: string; replyTo?: string; template: ReactElement }) {
  const html = await render(opts.template)          // async — always await it
  const text = toPlainText(html)
  return getTransporter().sendMail({ from: process.env.SMTP_FROM, to: opts.to, replyTo: opts.replyTo, subject: opts.subject, html, text })
}
```

- **Lazy, not module-scope** — a transporter created at import time runs during `next build`, where the env may not exist.
- **Fail loudly on missing config** — no `|| 'smtp.gmail.com'` (`smells.md` §3).
- **Node runtime only.** Nodemailer needs Node APIs — never call it from `proxy.ts` or an Edge route.
- `nodemailer` and `react-email` are server dependencies; suggest the installs, don't run them. `render()` returns a Promise — a missing `await` sends `[object Promise]` as the mail body.

### The action

```ts
// features/contact/actions/submit-contact.ts
'use server'

const ContactSchema = z.object({
  name: z.string().min(1).max(120),
  email: z.string().email(),
  message: z.string().min(10).max(5000),
  website: z.string().optional(),                 // honeypot
})

export async function submitContactAction(_prev: ActionResult | null, formData: FormData): Promise<ActionResult> {
  const parsed = ContactSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) return validationFailure(parsed.error)

  if (parsed.data.website) {
    // silent to the bot, visible to us: a real visitor tripping it would otherwise vanish
    console.warn('[contact] honeypot tripped')
    return { ok: true, data: undefined }
  }

  await rateLimitOrThrow('contact', await clientIp())   // the project's limiter

  try {
    await sendMail({
      to: requiredEnv('CONTACT_INBOX'),
      replyTo: parsed.data.email,                          // the visitor goes in replyTo, never from
      subject: `Contact form — ${parsed.data.name}`,
      template: <ContactNotification {...parsed.data} />,
    })
  } catch (error) {
    console.error('[mail] send failed', error)
    return { ok: false, error: { message: 'Your message could not be sent. Please try again or email us directly.', code: 'MAIL_FAILED' } }
  }
  return { ok: true, data: undefined }
}
```

(Write the action as `.tsx` if it renders the template inline.)

- **`from` is your authenticated domain, `replyTo` is the visitor.** Sending `from: visitor@gmail.com` fails DMARC and lands in spam — the most common contact-form bug.
- **Never leak the SMTP error** — provider hostnames and auth failures are internal. Log it, return a message the user can act on. This is the one place a frontend-facing message is written here: the failure is the server's own transport, not an upstream API response.
- **Rate-limit** every public send path, plus the honeypot.
- **Share `ContactSchema`** between the form and the action.

### Honeypot

```tsx
{/* hidden from people, still present for bots */}
<div className="absolute -left-[9999px]" aria-hidden="true">
  <label htmlFor="website">Leave this empty</label>
  <input id="website" name="website" type="text" tabIndex={-1} autoComplete="off" />
</div>
```

- **Validate it loosely and check it in the action.** `z.string().max(0)` returns a field error telling the bot exactly which field to leave alone. Accept any string, then return the same success a real send returns.
- **Silent to the bot, never silent to us.** Log each trip — if autofill starts filling it, real leads disappear behind a success message.
- **Pick a name no real field uses** (`website` is conventional; never `company`, `email`, `phone`).
- Off-screen rather than `display: none`, `tabIndex={-1}`, `aria-hidden`, `autoComplete="off"`.
- It stops dumb bots only; the rate limit is the second layer. Add Turnstile/CAPTCHA only if logs show spam getting past both.

**Verify the SMTP path early** — a dev inbox (Mailpit, Ethereal) in `.env.example` beats discovering on launch day that port 587 is blocked on the host.

### Diagnosing a send that doesn't arrive

First, did the action run? A form whose submit does nothing visible usually failed before the action (validation error rendered off-screen, or a client error). Then read the result and the server log:

| Result | Meaning | Next |
| :--- | :--- | :--- |
| `{ ok: true }` | The SMTP server **accepted** it | Inbox, then spam. Then SPF/DKIM/DMARC for the `from` domain. |
| `MAIL_FAILED` | The SMTP server **refused** it | The log line after `[mail] send failed` |
| `[CONFIG] Mail transport is not configured` | Env missing, or server not restarted since it was set | `smells.md` §3 |

- **`EAUTH` / 535** — wrong credentials: the **mailbox** password, full address as user.
- **Certificate / altname error** — use the server hostname the host's mail-client settings page lists, not `mail.<domain>`.
- **`ECONNREFUSED` / `ETIMEDOUT`** — port blocked from where the app runs; try the other of 465/587, or ask the host. Many serverless hosts block outbound SMTP entirely — an ESP's HTTP API is the fix there.

---

## 4. Slow sends and the request path

For a single transactional mail the user is waiting on ("we've sent your reset link"), awaiting it is correct. Beyond that:

- **Non-critical follow-ups** (a notification to the team after a save) can run after the response with `after()` from `next/server` — check that the deploy target supports it for the runtime used.
- **Never `await` a batch of sends in a Server Action or Route Handler.** It blocks the response and hits the platform's function timeout halfway through the batch, with no record of which half was sent. Use a queue or background job (the platform's queue, Inngest, Trigger.dev, a worker).
- **Make sends idempotent** — a send record keyed by (recipient, template, entity), checked first, or a retry doubles every mail.

---

## 5. Email marketing is not Nodemailer

If the requirement is a campaign to a list, say so plainly and don't build it on raw SMTP — you'd be rebuilding list management, bounces, complaints, suppression, unsubscribe and IP reputation, badly enough to get the domain blocklisted.

Use an ESP behind a Server Action / Route Handler. What stays your responsibility:

- **Consent recorded at signup** — timestamp, source, IP.
- **One-click unsubscribe** in every campaign and a `List-Unsubscribe` header — legally required in most jurisdictions.
- **Transactional and marketing separated** — different streams, ideally subdomains.
- **SPF, DKIM and DMARC** on the sending domain before the first campaign.
- **Never mail a list you didn't collect consent for**, and never scrape addresses from the CMS to build one. If asked to, say no and explain the exposure.

---

## 6. PDFs

Decide which case you have before writing code.

**(a) The PDF already exists (Strapi media, object storage).** Don't regenerate it. Link to it via `absoluteMediaUrl` (`strapi-next.md` §9). Proxy it through a Route Handler only if it must be access-controlled — and then the file must not also be publicly reachable.

**(b) Generated from data — the normal case.** A Route Handler renders and streams it:

```ts
// app/api/invoices/[id]/pdf/route.ts
import { renderToBuffer } from '@react-pdf/renderer'
import { getInvoiceForCurrentUser } from '@/features/billing/server/invoices'
import { InvoiceDocument } from '@/templates/pdf/invoice-document'

export const runtime = 'nodejs'

export async function GET(_req: Request, ctx: RouteContext<'/api/invoices/[id]/pdf'>) {
  const { id } = await ctx.params
  const invoice = await getInvoiceForCurrentUser(id)          // scoped by the session (auth.md)
  if (!invoice) return Response.json({ message: 'Invoice not found' }, { status: 404 })

  const pdf = await renderToBuffer(<InvoiceDocument invoice={invoice} />)
  return new Response(new Uint8Array(pdf), {
    headers: {
      'Content-Type': 'application/pdf',
      'Content-Disposition': `attachment; filename="invoice-${safeFilename(invoice.number)}.pdf"`,
      'Cache-Control': 'private, no-store',
    },
  })
}
```

(`route.tsx` if it renders JSX inline.)

- **Authorise before rendering**, scoped by the session — an unscoped `findUnique(id)` is the classic document-enumeration hole.
- **Sanitise the filename** — strip quotes, newlines and path separators, or it's header injection.
- **`inline` vs `attachment`** — pick deliberately.
- If the build or route fails with *"require() of ES Module not supported"*, add `serverExternalPackages: ['@react-pdf/renderer']` to `next.config` — a config change, so propose it rather than editing silently.
- The link is a plain `<a href="/api/invoices/…/pdf">`. Never fetch a PDF into a Client Component just to trigger a download.

**(c) HTML → PDF with a headless browser.** Puppeteer/Playwright costs ~300 MB of Chromium, seconds of cold start, and doesn't run on most serverless/edge targets. Reach for `@react-pdf/renderer` or `pdf-lib` first. If HTML-to-PDF is genuinely required, isolate it in its own service and say the trade-off out loud.

**Client-side generation** (`jsPDF`) only for something the user is already looking at (a chart export). Never for anything of record.

**Emailing a PDF:** attach the buffer directly (`attachments: [{ filename, content }]`) — serverless filesystems are read-only or ephemeral. Past a few MB, link to a signed URL instead.

---

## 7. Smells

1. **Mail credentials in two apps**, or in `NEXT_PUBLIC_*`.
2. **A transporter created per request** instead of the singleton.
3. **`from` set to the visitor's address** on a contact form.
4. **An HTML body built with a template literal** instead of a React Email component.
5. **A public send path with no rate limit and no honeypot.**
6. **Raw SMTP errors returned to the client.**
7. **A batch of sends awaited in an action or Route Handler.**
8. **A campaign path with no unsubscribe, no consent record, or transactional mail on the same stream.**
9. **A PDF Route Handler that trusts a client-supplied id** without scoping to the session.
10. **Puppeteer on a serverless deployment** for a document a template library could render.
11. **Nodemailer imported into `proxy.ts` or an Edge route.**
