# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# SG Couture Mobile

Mobile storefront client (second client, after the Next.js web storefront) for the standalone `sg-couture-api` NestJS backend at `/api/v1`. The app's job is disciplined contract consumption: correct auth modes, correct guest-cart identity, correct error-code handling, zero client-side re-derivation of business rules (prices, stock, discounts, totals all come from the server).

Read `docs/README.md`, `docs/00-architecture.md`, and `docs/01-conventions.md` before writing code. The storefront API guide in `docs/integration/storefront/` (`00-conventions.md` … `11-profile.md`) is the authoritative contract; if code or plan disagrees with it, stop and report the drift. **Ignore `docs/integration/admin/` entirely** — it documents the admin dashboard API, is not consumed by this app, and must never inform mobile code, types, or conventions.

## Current state

The repo is still the fresh `create-expo-app` template (Expo SDK 57, React Native 0.86, Expo Router). The target architecture below **does not exist yet** — Phase 0 (`docs/phase-0-foundation.md`) builds it. Template leftovers in `src/components/`, `src/hooks/`, `src/constants/` are placeholders, not conventions to follow. Axios, TanStack Query, Zustand, Clerk, expo-secure-store, and the test setup are not yet installed.

## Commands

Package manager is **bun** (`bun.lock`; no npm lockfile) — use `bun`/`bunx`, never npm/npx.

- `bun install` — install dependencies
- `bun run start` / `bun run ios` / `bun run android` / `bun run web` — start the Expo dev server
- `bun run lint` — `expo lint` (ESLint)
- `bunx tsc --noEmit` — typecheck (TS strict; path alias `@/*` → `./src/*`)

There are no `typecheck` or `test` scripts yet; Phase 0 adds them (Jest + `@testing-library/react-native`). Once they exist, every PR-sized change must pass `lint && typecheck && test`.

## Stack (fixed — do not substitute)

Expo (managed) · Expo Router · TypeScript strict · Axios · TanStack Query v5 · Zustand · @clerk/clerk-expo · expo-secure-store · bare `StyleSheet` + `src/theme/tokens.ts`. No styling libraries, no Redux, no fetch wrappers besides `src/lib/api/client.ts`.

## Target architecture (built in Phase 0, details in `docs/00-architecture.md`)

```
src/
├── app/        Expo Router routes ONLY — thin files
├── features/   ALL business/UI logic: <feature>/{api.ts, keys.ts, hooks.ts, components/, screens/}
├── lib/        api/{client,envelope,errors}.ts · query/queryClient.ts · storage/secureStorage.ts · format/money.ts
├── stores/     Zustand: cartSession.ts (guest token mirror), ui.ts
├── components/ shared primitives: Screen, Text, Button, Skeleton, EmptyState, ErrorState
├── theme/      tokens.ts — the only source of visual constants
└── strings/    strings.ts — all user-facing copy, incl. errorCode → message map
```

Data flow: screen → feature hook (TanStack Query) → feature `api.ts` → `lib/api/client.ts` (one axios instance; request interceptor attaches fresh Clerk token + `X-Cart-Session`, response interceptor unwraps the success envelope and throws typed `ApiError{status, code, message, errors[]}`).

Query conventions: hierarchical key factories in each feature's `keys.ts`; retries — never on 4xx, ≤2 on network/5xx, 0 for mutations; `staleTime` 60s for catalog, 0 for cart/orders; `useInfiniteQuery` driven by `meta.hasNext`/`meta.page` for products, reviews, orders.

## Hard rules

1. **Server state lives only in TanStack Query.** Never copy API payloads into Zustand/context. Cart mutations `setQueryData` with the returned full cart (do not invalidate-and-refetch — the response is authoritative).
2. **Branch on error `code`, never on `message`.** All error handling goes through `ApiError` and the narrowers in `src/lib/api/errors.ts`. Global handlers (registered once in the query client): `UNAUTHENTICATED` → re-auth prompt, `ACCOUNT_DISABLED` → sign out + dedicated screen, `RATE_LIMITED` → toast, no retry.
3. **Money strings:** decimal strings with variable scale; display via `lib/format/money.ts`; no client-side arithmetic on money; never `parseFloat` for logic.
4. **Route files in `src/app/` are thin:** parse params → render a screen from `src/features/*/screens`. No fetching, logic, or styling there.
5. **Tokens are secrets:** never log, persist outside SecureStore, or place in URLs/analytics the Clerk JWT, guest `sessionToken`, or order claim tokens.
6. **Fresh Clerk token per request** via `getToken()` in the interceptor; never cache it.
7. **Guest cart lifecycle:** capture `sessionToken` from the first anonymous `POST /cart/items`; delete it after merge (first authed cart response), guest checkout success, or anonymous clear. There is no merge endpoint — `GET /cart` with both identities attached *is* the merge.
8. **204 routes return no body** — never JSON-parse them (cart clear, wishlist remove, address delete, review delete).
9. **Throttled routes** (`/orders`, `/orders/guest`, `/orders/claim`, `/coupons/validate`, guest tracking): mutations never auto-retry; submit buttons disable in flight; 429 preserves form state.
10. **No polling loops.** Refresh on screen focus (focusManager + AppState) and pull-to-refresh only.
11. **CASH only.** `CARD` stays a disabled option behind the payment-method config; `POST /orders/:id/payment-session` does not exist.
12. **Styling:** `StyleSheet.create`, values from `theme/tokens.ts` only — zero raw hex/px literals in components. User-facing copy comes from `src/strings/strings.ts`.
13. **IDs are opaque.** Display `humanOrderId`; use `id` in routes/API. Category/product filters use **slugs**.

## Workflow

- Execute phases in order from `docs/phase-0-foundation.md` … `docs/phase-7-hardening-release.md`; a phase's Definition of Done gates the next.
- Before implementing an endpoint, re-read its module doc in `docs/integration/storefront/` — validation rules, error tables, and structured `errors[]` shapes are all there.
- New architectural decisions get a short ADR following the mini-ADR format in `docs/00-architecture.md` (ADR-M001…M005).
- Prefer editing the canonical feature template shape (`features/products/` after Phase 1) over inventing new structures for later features.
- Naming: files `kebab-case.ts`, components `PascalCase.tsx`; no barrel files inside `features/*` except a deliberate `index.ts` exporting screens + public hooks.
