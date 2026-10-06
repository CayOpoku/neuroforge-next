# Testing

Load when writing or fixing tests. Test the things that break; do not chase a coverage number.

---

## 1. Setup

```bash
pnpm add -D vitest @vitejs/plugin-react vite-tsconfig-paths jsdom @testing-library/react @testing-library/dom @testing-library/user-event
pnpm add -D @playwright/test   # end-to-end, for Server Components and full flows
```

```ts
// vitest.config.mts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import tsconfigPaths from 'vite-tsconfig-paths'

export default defineConfig({
  plugins: [tsconfigPaths(), react()],
  test: { environment: 'jsdom' },
})
```

Scripts: `test` → `vitest run`, `test:watch` → `vitest`, `test:e2e` → `playwright test`. Suggest installs; don't run them unprompted.

**Async Server Components aren't supported by unit-test renderers.** Test their data functions directly, and cover the rendered page with Playwright.

---

## 2. What is worth testing

| Priority | Target | Why |
| :--- | :--- | :--- |
| High | Data-access and service functions (`features/*/server/`) | Business rules and scoping live here; no rendering needed |
| High | Validation schemas — valid, invalid, edge input | The contract every form and action depends on |
| High | Auth: `requireUser`, `requireRole`, tenant scoping, Server Actions rejecting other tenants' ids | A silent failure here is a breach |
| High | `getErrorMessage` and `fetchJson` | Every error the user sees passes through them |
| Medium | Server Actions end-to-end (auth → validation → result shape) | Called as plain async functions with the session mocked |
| Medium | Client Components with branching state (forms, wizards, tables) | Fast with Testing Library |
| Medium | Critical flows — sign-up, checkout, create → list | Playwright against `next build && next start` |
| Low | Presentational components | Slow, brittle, low yield |
| Never | shadcn primitives, Next.js behaviour | Not your code |

If a bug reaches production, a regression test for it is mandatory.

---

## 3. Patterns

```ts
// unit — pure logic
import { describe, expect, it } from 'vitest'
import { ApiError } from '@/lib/api'
import { getErrorMessage } from '@/lib/error'

describe('getErrorMessage', () => {
  it('reads a FastAPI detail array', () => {
    expect(getErrorMessage(new ApiError(422, { detail: [{ msg: 'Field required', loc: ['body', 'email'] }] })))
      .toBe('Field required')
  })

  it('reads a failed ActionResult', () => {
    expect(getErrorMessage({ ok: false, error: { message: 'Tenant quota exceeded' } })).toBe('Tenant quota exceeded')
  })

  it('falls back to a transport message when the request never reached the server', () => {
    expect(getErrorMessage(new TypeError('Failed to fetch')))
      .toBe('Could not reach the server. Check your connection and try again.')
  })
})
```

```ts
// Server Action — called directly, session mocked
import { vi, it, expect } from 'vitest'

vi.mock('server-only', () => ({}))
vi.mock('@/server/auth', () => ({ requireUser: vi.fn().mockResolvedValue({ id: 'u1', organizationId: 'org1' }) }))
vi.mock('next/cache', () => ({ updateTag: vi.fn(), revalidateTag: vi.fn() }))

it('returns field errors instead of throwing on invalid input', async () => {
  const { createOrderAction } = await import('@/features/orders/actions/create-order')
  const form = new FormData()
  form.set('email', 'nope')
  const result = await createOrderAction(null, form)
  expect(result).toMatchObject({ ok: false, error: { code: 'VALIDATION' } })
})
```

```tsx
// Client Component
import { render, screen } from '@testing-library/react'

it('renders the error state instead of a fallback value', () => {
  render(<OrderStatus status="error" error={new Error('nope')} />)
  expect(screen.queryByText('Pending')).toBeNull()
  expect(screen.getByRole('alert')).toBeTruthy()
})
```

```ts
// e2e — a real request through the built app
import { test, expect } from '@playwright/test'

test('rejects another tenant’s invoice with 404', async ({ page }) => {
  await page.goto('/invoices/00000000-0000-0000-0000-000000000000')
  await expect(page.getByRole('heading', { name: /not found/i })).toBeVisible()
})
```

---

## 4. Rules

- One behaviour per test. The test name states the behaviour, not the function name.
- Independent and repeatable: no shared mutable state, no reliance on order, no real network or third-party calls.
- Assert the **contract**, not the implementation.
- Test the failure paths — invalid input, unauthenticated, another tenant's id, empty list, network error. Those are the ones that ship broken.
- Run e2e against a production build — `next dev` hides caching and bundling behaviour.
- Never weaken an assertion to make a test pass. Fix the code, or delete the test and say why.
