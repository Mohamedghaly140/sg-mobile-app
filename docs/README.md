# SG Couture Mobile — Development Plan

> **Audience:** Claude Code (and human reviewers) implementing the SG Couture mobile app.
> **Stack:** Expo (React Native) · Expo Router · TypeScript · Axios · TanStack Query v5 · Zustand · Clerk Expo · bare `StyleSheet` with design tokens.
> **Backend contract:** the storefront API integration guide (`docs/integration/storefront/00-conventions.md` … `11-profile.md`). The server is the source of truth for prices, stock, discounts, shipping, and totals. **This plan never re-derives business rules — it references them.**

## How to use this plan

1. Read [`00-architecture.md`](./00-architecture.md) and [`01-conventions.md`](./01-conventions.md) once. Every phase assumes them.
2. Execute phases **in order**. Each phase doc is a self-contained work order: objective, prerequisites, task breakdown, error handling, and acceptance criteria.
3. Do not start a phase until the previous phase's **Definition of Done** is met. Phases are sequenced so that each one ships a testable vertical slice.
4. When a phase doc and the backend API guide disagree, **the API guide wins** — flag the drift instead of guessing.

## Phase map

| Phase | Doc | Delivers | API modules |
|---|---|---|---|
| 0 | [phase-0-foundation.md](./phase-0-foundation.md) | Project scaffold, API client, query/store infrastructure, theme tokens, navigation shell | 00 (conventions) |
| 1 | [phase-1-catalog.md](./phase-1-catalog.md) | Browsable catalog: home, category tree, product list + filters, product detail, public reviews | 01, 02, 03 (read) |
| 2 | [phase-2-guest-cart.md](./phase-2-guest-cart.md) | Full guest cart with `X-Cart-Session` lifecycle | 05 |
| 3 | [phase-3-auth-cart-merge.md](./phase-3-auth-cart-merge.md) | Clerk auth, token interceptor, anonymous→user cart merge, profile | 00 (auth), 11 |
| 4 | [phase-4-account.md](./phase-4-account.md) | Wishlist, own-review CRUD, saved addresses | 03 (write), 04, 08 |
| 5 | [phase-5-checkout.md](./phase-5-checkout.md) | Shipping estimate, coupon preview, registered + guest checkout (CASH) | 06, 07, 09 |
| 6 | [phase-6-orders.md](./phase-6-orders.md) | Order history/detail, guest tracking, claim via deep link, self-cancel | 10 |
| 7 | [phase-7-hardening-release.md](./phase-7-hardening-release.md) | Performance, resilience, accessibility, EAS build/release, Geidea readiness | — |

## Non-goals (v1)

- **Card payments.** Backend gates `CARD` with `422 PAYMENT_METHOD_UNAVAILABLE` until backend Phase 7 (Geidea). The UI ships CASH-only behind a payment-method abstraction so CARD can be enabled without restructuring (see phase 5 + 7).
- **Push / in-app notifications.** Backend Phase 9. Order status is communicated by transactional email; the app refreshes on screen focus, never polls in a loop.
- **Localization.** English only in v1. All user-facing strings still go through a single `strings.ts` module so i18n can be added later without a codemod hunt.
- **Anonymous wishlist sync.** Wishlist is auth-only per the API; guests see a sign-in prompt.

## Working agreements

- Features-first structure; route files in `src/app/` stay thin and delegate to `src/features/*` (see architecture doc).
- Server state lives **only** in TanStack Query. Zustand holds client state (cart session token mirror, UI state, filter drafts). No duplication of server data into stores.
- Branch on error `code`, never on `message`.
- Money values are decimal **strings** with variable scale — format for display, never do client-side arithmetic on them (details in conventions).
- Every phase ends with the manual test checklist in its Definition of Done executed on iOS and Android.
