# 00 — Architecture

## Context and goals

SG Couture mobile is the second client of the standalone `sg-couture-api` NestJS backend (the Next.js storefront is the first). The API is a **live, versioned contract** (`/api/v1`) designed for "one backend, N clients", so the app's job is disciplined consumption: correct auth modes, correct guest-cart identity, correct error-code handling, and zero client-side re-derivation of business rules.

Goals, in priority order:

1. **Contract fidelity** — every screen behaves exactly as the API guide describes, including edge codes (`INSUFFICIENT_STOCK` line errors, `CLAIM_TOKEN_INVALID`, merge semantics).
2. **Maintainability** — features-first modules with a thin shared core, mirroring the team's established clean-architecture habits from ORTH.
3. **Perceived performance** — cache-first reads via TanStack Query, optimistic updates only where the API is idempotent, skeletons over spinners.
4. **Forward compatibility** — CARD/Geidea (backend Phase 7) and notifications (backend Phase 9) plug in without restructuring.

## Stack decisions (mini-ADRs)

### ADR-M001 — Expo managed workflow + Expo Router

**Accepted.** Expo managed workflow with EAS Build; Expo Router for file-based navigation.

- *Why:* fastest iteration for a two-platform storefront; Expo Router gives typed routes, deep linking (needed for the guest-order claim email link) and web-shareable URL semantics for free; no native module needs identified in v1 that require prebuild ejection (Clerk, SecureStore, Image are all Expo-compatible).
- *Trade-off:* file-based routing constrains where screens live. Mitigated by the "thin route file" rule below.

### ADR-M002 — TanStack Query owns server state; Zustand owns client state

**Accepted.**

- TanStack Query v5 is the **only** cache for anything the API returns. No API payload is ever copied into a Zustand store or context.
- Zustand holds: the cart session token mirror (hydrated from SecureStore), transient UI state (active filter draft on the products screen, checkout step), and cross-screen ephemeral flags.
- *Why:* eliminates the classic dual-source-of-truth drift; cart mutations return the full cart, which maps perfectly to `setQueryData`.
- *Consequence:* "is the cart badge correct?" is always answered by the cart query cache, never a store.

### ADR-M003 — Axios with a single configured instance + interceptors

**Accepted.** One `apiClient` instance:

- Request interceptor attaches a **fresh Clerk token per request** (`getToken()` — the API guide forbids caching the session JWT) and the `X-Cart-Session` header when a guest token exists.
- Response interceptor unwraps the success envelope and normalizes failures into a typed `ApiError` (`status`, `code`, `message`, `errors[]`).
- *Why:* every feature's API module then deals only in domain payloads and typed errors — the RN analogue of ORTH's `ApiResult<T>` discipline, adapted to TanStack Query's thrown-error model.

### ADR-M004 — Bare `StyleSheet` + design tokens module

**Accepted.** No styling library. A `src/theme/tokens.ts` module (colors, spacing, radii, typography scale) is the only source of visual constants; real token values arrive later from the design system and drop into this one file. Components never hardcode raw values.

### ADR-M005 — Clerk Expo for auth

**Accepted** (dictated by backend ADR-0001). `@clerk/clerk-expo` with `tokenCache` backed by `expo-secure-store`. The backend issues no tokens of its own; all three auth modes (Public / Optional / Auth) are handled purely by whether the interceptor can attach a Clerk token.

## Repository layout

```
sg-mobile-app/
├── app.json / eas.json
├── CLAUDE.md                     ← rules for Claude Code (repo root)
├── docs/
│   ├── integration/storefront/   ← the backend integration guide (copied, read-only)
│   └── *.md                      ← this plan (README, architecture, conventions, phase docs)
└── src/
    ├── app/                      ← Expo Router routes ONLY (thin files)
    │   ├── _layout.tsx           ← providers: Clerk → QueryClient → theme
    │   ├── (tabs)/
    │   │   ├── _layout.tsx       ← tab bar: Home, Categories, Cart, Account
    │   │   ├── index.tsx         ← home / featured
    │   │   ├── categories.tsx
    │   │   ├── cart.tsx
    │   │   └── account.tsx
    │   ├── products/
    │   │   ├── index.tsx         ← list (accepts filter params)
    │   │   └── [slug].tsx        ← product detail
    │   ├── checkout/             ← stack: address → review → confirm
    │   ├── orders/
    │   │   ├── index.tsx
    │   │   ├── [id].tsx
    │   │   └── track/[token].tsx ← public guest tracking (deep link target)
    │   ├── (auth)/               ← sign-in / sign-up (Clerk)
    │   └── addresses/ …
    ├── features/                 ← ALL business/UI logic lives here
    │   ├── products/
    │   │   ├── api.ts            ← axios calls + request/response types
    │   │   ├── keys.ts           ← query key factory
    │   │   ├── hooks.ts          ← useProducts, useProduct, …
    │   │   ├── components/       ← ProductCard, FilterSheet, …
    │   │   └── screens/          ← ProductListScreen, ProductDetailScreen
    │   ├── categories/
    │   ├── reviews/
    │   ├── wishlist/
    │   ├── cart/
    │   ├── coupons/
    │   ├── shipping/
    │   ├── addresses/
    │   ├── checkout/
    │   ├── orders/
    │   └── profile/
    ├── lib/
    │   ├── api/
    │   │   ├── client.ts         ← axios instance + interceptors
    │   │   ├── envelope.ts       ← success envelope types + unwrap
    │   │   └── errors.ts         ← ApiError, code constants, type guards
    │   ├── query/
    │   │   └── queryClient.ts    ← defaults: retry policy, staleTime, gcTime
    │   ├── storage/
    │   │   └── secureStorage.ts  ← typed expo-secure-store wrapper
    │   └── format/
    │       └── money.ts          ← decimal-string display formatting
    ├── stores/
    │   ├── cartSession.ts        ← guest session token mirror + lifecycle actions
    │   └── ui.ts
    ├── components/               ← shared presentational (Button, Skeleton, EmptyState, ErrorState, Screen)
    ├── theme/
    │   └── tokens.ts
    └── strings/
        └── strings.ts            ← all user-facing copy (error-code → message map lives here)
```

**Thin route file rule:** files under `src/app/` may only: parse route params, pick a screen from `src/features/*/screens`, and render it. No data fetching, no business logic, no styles beyond layout glue. This keeps Expo Router's file constraints from dictating architecture.

## Data flow

```
Screen (feature/screens)
  → feature hook (useCart, useProducts…)          TanStack Query
    → feature api.ts function                     typed request
      → lib/api/client.ts (interceptors)          Clerk token + X-Cart-Session
        → backend /api/v1
      ← envelope unwrapped → typed payload  |  thrown ApiError{code, errors[]}
    ← query/mutation cache update (setQueryData / invalidate)
  ← render from cache; errors mapped via strings/errorMessages
```

Three flows deserve names because multiple phases depend on them:

1. **Guest cart identity flow** — first anonymous `POST /cart/items` returns `sessionToken` once → persist to SecureStore → hydrate `cartSession` store on app start → interceptor attaches `X-Cart-Session` on every cart-aware call. Token is **deleted** on: successful merge (first authenticated cart response), successful guest checkout, anonymous cart clear. (Owned by Phase 2; consumed by 3, 5.)
2. **Merge flow** — immediately after Clerk sign-in resolves, call `GET /cart` **while the stored guest header is still attached**, then delete the stored token and `setQueryData` the returned merged cart. There is no merge endpoint; this GET *is* the merge. (Phase 3.)
3. **Claim deep-link flow** — email link → `orders/track/[token]` (public detail via `GET /orders/guest/:token`) → "Add to my account" → sign-in → `POST /orders/claim` → token consumed, invalidate order list. (Phase 6.)

## State ownership matrix

| State | Owner | Persistence |
|---|---|---|
| Products, categories, reviews, cart, wishlist, addresses, orders, profile | TanStack Query cache | in-memory (optionally persisted later; not v1) |
| Clerk session | Clerk SDK | SecureStore (Clerk tokenCache) |
| Guest cart session token | `cartSession` Zustand store | SecureStore (source of truth) |
| Filter/sort draft, checkout step, sheet visibility | `ui` Zustand store / local state | none |
| Design tokens | `theme/tokens.ts` | code |

## Integration points & failure domains

- **Clerk outage / token failure** → interceptor proceeds tokenless on Optional routes; Auth routes surface `UNAUTHENTICATED` → global handler routes to sign-in.
- **`ACCOUNT_DISABLED` (403)** → global handler: sign out via Clerk, show the account-disabled screen. Handled once in `errors.ts` consumers, not per feature.
- **Rate limits (429)** → mutations never auto-retry; checkout/claim buttons disable while in flight (API guide mandates this for the 5/60s routes).
- **Stale money/stock** → all catalog `quantity`/price values are display hints; checkout errors (`INVALID_VARIANT`, `INSUFFICIENT_STOCK`) are the authoritative reconciliation moment (Phase 5).

## What we'll revisit

- Query cache persistence (`@tanstack/query-persist-client`) once offline behavior is prioritized (Phase 7 evaluates, doesn't commit).
- `CARD` payment UI + Geidea SDK/session flow when backend Phase 7 ships — the `PaymentMethod` abstraction in Phase 5 is the seam.
- Notifications module when backend Phase 9 ships.
