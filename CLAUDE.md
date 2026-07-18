# CLAUDE.md — SG Couture Mobile

Rules for Claude Code working in this repository. Read `docs/dev-plan/README.md`, `00-architecture.md`, and `01-conventions.md` before writing code. The backend API guide in `docs/api/` is the authoritative contract; if code or plan disagrees with it, stop and report the drift.

## Stack (fixed — do not substitute)

Expo (managed) · Expo Router · TypeScript strict · Axios · TanStack Query v5 · Zustand · @clerk/clerk-expo · expo-secure-store · bare `StyleSheet` + `src/theme/tokens.ts`. No styling libraries, no Redux, no fetch wrappers besides `src/lib/api/client.ts`.

## Hard rules

1. **Server state lives only in TanStack Query.** Never copy API payloads into Zustand/context. Cart mutations `setQueryData` with the returned full cart.
2. **Branch on error `code`, never on `message`.** All error handling goes through `ApiError` and the narrowers in `src/lib/api/errors.ts`.
3. **Money strings:** display via `lib/format/money.ts`; no client-side arithmetic on money; never `parseFloat` for logic.
4. **Route files in `src/app/` are thin:** parse params → render a screen from `src/features/*/screens`. No fetching, logic, or styling there.
5. **Tokens are secrets:** never log, persist outside SecureStore, or place in URLs/analytics the Clerk JWT, guest `sessionToken`, or order claim tokens.
6. **Fresh Clerk token per request** via `getToken()` in the interceptor; never cache it.
7. **Guest cart lifecycle:** capture `sessionToken` from the first anonymous `POST /cart/items`; delete it after merge (first authed cart response), guest checkout success, or anonymous clear. There is no merge endpoint — `GET /cart` with both identities *is* the merge.
8. **204 routes return no body** — never JSON-parse them (cart clear, wishlist remove, address delete, review delete).
9. **Throttled routes** (`/orders`, `/orders/guest`, `/orders/claim`, `/coupons/validate`, guest tracking): mutations never auto-retry; submit buttons disable in flight; 429 preserves form state.
10. **No polling loops.** Refresh on screen focus (focusManager + AppState) and pull-to-refresh only.
11. **CASH only.** `CARD` stays a disabled option behind the payment-method config; `POST /orders/:id/payment-session` does not exist.
12. **Styling:** `StyleSheet.create`, values from `theme/tokens.ts` only — zero raw hex/px literals in components. User-facing copy comes from `src/strings/strings.ts`.
13. **IDs are opaque.** Display `humanOrderId`; use `id` in routes/API. Category/product filters use **slugs**.

## Workflow

- Execute phases in order from `docs/dev-plan/phases/`; a phase's Definition of Done gates the next.
- Before implementing an endpoint, re-read its module doc in `docs/api/` — validation rules, error tables, and structured `errors[]` shapes are all there.
- New architectural decisions get a short ADR in `docs/dev-plan/adr/` following the existing mini-ADR format.
- Every PR-sized change: `npm run lint && npm run typecheck && npm run test` must pass.
- Prefer editing the canonical feature template shape (`features/products/` after Phase 1) over inventing new structures for later features.
