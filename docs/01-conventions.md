# 01 — App Conventions (read before any phase)

Companion to the backend guide's `00-conventions.md`. That doc defines the wire contract; this one defines how this codebase consumes it. If they conflict, the backend doc wins.

## 1. API client

One axios instance in `src/lib/api/client.ts`:

- `baseURL` = `${EXPO_PUBLIC_API_URL}/api/v1` from env (`.env` → `app.config.ts`). Never hardcode hosts.
- **Request interceptor**
  - Fetch a **fresh** Clerk token per request via `getToken()` and set `Authorization: Bearer …` when signed in. Never cache the JWT (contract requirement).
  - If `cartSession` store holds a guest token, set `X-Cart-Session`. Attach it even when authenticated — that overlap is exactly what triggers the server-side merge.
- **Response interceptor**
  - `204` → resolve `undefined`. Never attempt JSON parsing on documented-204 routes (cart clear, wishlist remove, address delete, review delete).
  - Success envelope → return `{ data, meta? }` unwrapped.
  - Error → throw `ApiError`.

## 2. Errors

`src/lib/api/errors.ts`:

```ts
export class ApiError extends Error {
  constructor(
    public status: number,
    public code: ErrorCode | string, // branch on this, never on message
    message: string,
    public errors?: unknown[],       // structured lines when documented
  ) { super(message); }
}
```

- `ErrorCode` is a union of every documented code (generic block + storefront block). Unknown codes fall through to a generic failure message — the API may add codes.
- Typed narrowers for the two structured shapes:
  - `getStockErrors(e): { productId; requested; available }[]` for `INSUFFICIENT_STOCK`
  - `getVariantErrors(e): { productId; color; size; code }[]` for `INVALID_VARIANT`
  - `getValidationErrors(e): { field; constraints }[]` for `VALIDATION_ERROR` (map dotted paths like `contact.email` to form fields)
- **Global handlers** (registered once, in the query client's error hooks):
  - `401 UNAUTHENTICATED` on an Auth route → prompt re-auth via Clerk, retry after.
  - `403 ACCOUNT_DISABLED` → sign out + dedicated screen.
  - `429 RATE_LIMITED` → toast with backoff messaging; never auto-retry mutations.
- **Copy** lives in `src/strings/strings.ts` as an `errorCode → user message` map (one distinct message per coupon code, stock, variant, etc. — the API guide explicitly requires distinct copy per code).

## 3. TanStack Query

Defaults in `src/lib/query/queryClient.ts`:

| Setting | Value | Reason |
|---|---|---|
| `retry` | custom: never retry `ApiError` with status 4xx; retry ≤ 2 for network/5xx | 4xx are contract answers, not transience |
| `staleTime` | 60s catalog reads; 0 for cart/orders. Near-static data may override upward per feature (categories use 5 min — Phase 1) | catalog counts/stock are hints anyway |
| `refetchOnWindowFocus` | on (screen focus via `focusManager` + AppState) | replaces polling — orders doc forbids polling loops |
| mutations `retry` | 0 | throttled routes must not be hammered |

**Query key factories** — one `keys.ts` per feature, hierarchical:

```ts
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: ProductFilters) => [...productKeys.lists(), filters] as const,
  detail: (slug: string) => [...productKeys.all, 'detail', slug] as const,
};
```

Same pattern: `categoryKeys`, `reviewKeys(productId)`, `cartKeys.current`, `wishlistKeys.all`, `addressKeys`, `orderKeys.list(filters)` / `orderKeys.detail(id)` / `orderKeys.guest(token)`, `profileKeys.me`.

**Cache update rules:**

- Cart mutations return the **complete cart** → `queryClient.setQueryData(cartKeys.current, cart)`. Do not invalidate-and-refetch; the response is authoritative.
- Wishlist add/remove are idempotent → optimistic update with rollback on error.
- Review create/update/delete → invalidate `reviewKeys(productId)` **and** `productKeys.detail(slug)` (aggregates recomputed server-side).
- Checkout success → `setQueryData(cartKeys.current, emptyCart)` + invalidate `orderKeys.list`.
- Pagination: `useInfiniteQuery` driven by `meta.hasNext` / `meta.page` for products, reviews, orders. Categories, wishlist, addresses, cart items are unpaginated — plain queries.

## 4. Money, dates, IDs

- Money is a **decimal string with variable scale** (`"2400"` vs `"65.00"`). Rules:
  - Display through `lib/format/money.ts` only (`formatMoney("2040.5") → "EGP 2,040.50"`).
  - **No client arithmetic on money.** Totals, discounts, and fees always come from the server response being displayed. If a UI ever needs derived money (it shouldn't in v1), that's a design smell — surface it.
  - Never `parseFloat` for anything other than final display formatting.
- Dates: ISO UTC strings → format with a single `formatDate` helper; render in device locale/timezone.
- IDs: treat all IDs as opaque strings. Display `humanOrderId` to users; use `id` in routes/API calls (orders doc).

## 5. Auth modes in practice

| Mode | App behavior |
|---|---|
| Public | Call freely; token attachment is harmless |
| Optional | Call freely; interceptor decides what identity rides along |
| Auth | Screen is gated: signed-out users see an inline sign-in prompt component (`<RequireAuth>`), not a hard redirect, so guests can still browse tabs |

## 6. Styling

- `StyleSheet.create` only; styles co-located with the component.
- All values from `theme/tokens.ts` (placeholder scale now; real design-system tokens drop in later). No raw hex/px literals in components.
- Shared primitives in `src/components/`: `Screen`, `Text` (typography variants), `Button` (loading/disabled states built in — throttled routes depend on this), `Skeleton`, `EmptyState`, `ErrorState` (takes an `ApiError`, renders mapped copy + retry).

## 7. Naming & structure

- Files: `kebab-case.ts`; components `PascalCase.tsx`; hooks `useThing.ts` exported from feature `hooks.ts`.
- Feature API functions named after the endpoint intent: `getProducts`, `addCartItem`, `claimGuestOrder`.
- Types: request DTOs `AddCartItemBody`, responses `Cart`, `Product`, `OrderDetail`, `OrderSummary` — defined next to the api.ts that uses them; shared shapes promoted only when a second feature needs them.
- No barrel files inside `features/*` except a deliberate `index.ts` exporting the screens + public hooks.

## 8. Testing baseline (applies to every phase)

- Unit: `errors.ts` narrowers, `money.ts`, `cartSession` store lifecycle, per-feature api.ts response mapping (mocked axios).
- Hooks: TanStack Query hooks with `@testing-library/react-native` + a test QueryClient (retries off).
- Each phase's DoD lists the manual device checklist; automated coverage targets the logic above, not screenshots.
