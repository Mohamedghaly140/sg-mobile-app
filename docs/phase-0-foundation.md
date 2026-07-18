# Phase 0 — Foundation

**Objective:** a running Expo app with the full infrastructure skeleton — navigation shell, API client with interceptors, query client, stores, theme tokens, shared components — such that every later phase only adds feature modules.

**Prerequisites:** backend reachable in a dev environment; Clerk publishable key (test instance); `docs/integration/storefront/` copied into the repo.

**API surface:** none consumed yet beyond a smoke call; this phase implements everything `01-conventions.md` §1–§3 describes.

## Tasks

### 0.1 Project scaffold
- [ ] `create-expo-app` (latest SDK, TypeScript template), strict `tsconfig`, path alias `@/* → src/*`.
- [ ] ESLint + Prettier (expo config base); `bun run lint`, `typecheck`, `test` scripts.
- [ ] `app.config.ts` reading `EXPO_PUBLIC_API_URL`, `EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY`; `.env.example` committed.
- [ ] Directory skeleton exactly as `00-architecture.md` — create empty feature folders with `index.ts` placeholders so structure is enforced from day one.

### 0.2 Theme + shared components
- [ ] `theme/tokens.ts` with placeholder scales: `colors`, `spacing (4-pt)`, `radii`, `typography` variants. Single export, typed `as const`. (Real design-system values replace contents later — shape must not change.)
- [ ] Shared components: `Screen`, `Text`, `Button` (variants + `loading` prop that disables and shows spinner), `Skeleton`, `EmptyState`, `ErrorState({ error, onRetry })`.
- [ ] `strings/strings.ts` with the initial `errorMessages: Record<ErrorCode, string>` covering the full generic + storefront code tables from the API conventions doc.

### 0.3 API layer
- [ ] `lib/api/envelope.ts` — `SuccessEnvelope<T>`, `PaginationMeta`, `Paginated<T> = { data: T[]; meta: PaginationMeta }`.
- [ ] `lib/api/errors.ts` — `ApiError`, `ErrorCode` union, `isApiError`, `getStockErrors`, `getVariantErrors`, `getValidationErrors`. Unit tests against the exact JSON shapes in the API conventions doc.
- [ ] `lib/api/client.ts` — axios instance; request interceptor (Clerk token via injected `getToken` — wire the actual function in Phase 3, accept an injectable provider now; `X-Cart-Session` from `cartSession` store); response interceptor (204 → `undefined`; unwrap envelope; normalize to `ApiError`, including non-envelope failures like network errors → synthetic `NETWORK_ERROR` code).
- [ ] `lib/format/money.ts` + `formatDate` with unit tests (`"2400"`, `"2040.5"`, `"65.00"` all format correctly).

### 0.4 State infrastructure
- [ ] `lib/query/queryClient.ts` with the retry/staleTime/focus policies from conventions §3; wire `focusManager` to `AppState`.
- [ ] `stores/cartSession.ts` — `{ token: string | null }` + actions `hydrate()` (SecureStore read at app start), `setToken(t)` (store + persist), `clear()` (store + SecureStore delete). Unit-test the lifecycle with a mocked SecureStore.
- [ ] `lib/storage/secureStorage.ts` typed wrapper (`getItem/setItem/deleteItem` with key constants: `CART_SESSION`, plus Clerk's own cache handled by Clerk).
- [ ] `stores/ui.ts` empty shell (grows per phase).

### 0.5 Navigation shell + providers
- [ ] `src/app/_layout.tsx`: `ClerkProvider` (tokenCache via SecureStore) → `QueryClientProvider` → gesture/status-bar chrome. Call `cartSession.hydrate()` before first render (splash gate).
- [ ] `(tabs)` layout with 4 placeholder tabs (Home, Categories, Cart, Account) rendering `EmptyState`s.
- [ ] Deep-link scheme configured in `app.config.ts` (`sgcouture://`) — route table reserved for `orders/track/[token]` (used in Phase 6).

### 0.6 Smoke verification
- [ ] Temporary dev screen calls `GET /categories` through the full stack (client → envelope → query hook) and renders names; proves interceptors + envelope + query client end-to-end. Removed at phase end (Phase 1 replaces it).

## Definition of Done
- App boots on iOS + Android simulators; tabs navigate.
- `lint`, `typecheck`, and unit tests green.
- Smoke call renders live category names; killing the network shows `ErrorState` with retry working.
- No component contains a raw color/spacing literal.

## Out of scope
Any real screen, auth UI, cart logic.
