# Phase 2 — Guest Cart

**Objective:** full cart functionality for anonymous users with a correct `X-Cart-Session` lifecycle. This phase is the riskiest contract surface in the app — implement it exactly as the API guide's guest-identity section describes.

**Prerequisites:** Phase 1 DoD (product detail supplies the variant selection).

**API surface:** `GET /cart`, `POST /cart/items`, `PATCH /cart/items/:itemId`, `DELETE /cart/items/:itemId`, `DELETE /cart`. All **Optional** auth.

## Session token lifecycle (the contract in one list)

1. App start: `cartSession.hydrate()` (done in Phase 0).
2. First anonymous `POST /cart/items` → response `data` contains `sessionToken` **once**. Persist via `cartSession.setToken()` **before any navigation** — the API doc warns mobile must not lose it. Implement in the mutation's `onSuccess` synchronously.
3. Every subsequent cart-aware call carries `X-Cart-Session` (interceptor, already built).
4. `DELETE /cart` while anonymous → server deletes the cart row → `cartSession.clear()`.
5. Never log the token, never put it in a URL or analytics event.
6. Anonymous carts expire 7 days after the last mutation (`expiresAt` in every cart response) — surface subtly (e.g., nothing in v1 UI, but on a 404-ish empty cart after expiry the app just shows the empty state; no special handling needed because `GET /cart` returns a virtual empty cart).

## Tasks

### 2.1 Cart feature core
- [ ] Types: `Cart`, `CartItem` (line `price` = creation-time snapshot; `product.priceAfterDiscount` = live — both displayed per 2.3), virtual-empty-cart shape (`id: null`).
- [ ] `api.ts`: five endpoint functions. `addCartItem` returns `Cart & { sessionToken?: string }`.
- [ ] `useCart()` (staleTime 0, refetch on focus); mutations `useAddCartItem`, `useUpdateCartItem`, `useRemoveCartItem`, `useClearCart`.
- [ ] **Cache rule:** every mutation except clear returns the complete cart → `setQueryData(cartKeys.current, cart)`. Clear (204) → `setQueryData` to the virtual empty cart and, if anonymous, `cartSession.clear()`.
- [ ] No optimistic updates on cart mutations in v1 — stock validation happens server-side and the full-cart response is immediate truth. Use per-line pending indicators instead.

### 2.2 Add to cart (product detail)
- [ ] "Add to cart" button on product detail: requires size/color selection when the product has those arrays (client-side guard mirrors `INVALID_VARIANT` semantics); quantity stepper (min 1 — zero uses delete, per validation rules).
- [ ] `onSuccess`: persist `sessionToken` if present (first mutation), toast + cart badge update.
- [ ] `INSUFFICIENT_STOCK` (409): read `errors[]` via `getStockErrors`, show "only N available" on the quantity control using the `available` value — never parse the human message.
- [ ] `INVALID_VARIANT` (422): refresh product detail (arrays changed server-side), prompt reselect.
- [ ] `RESOURCE_NOT_FOUND` (404): product no longer ACTIVE → unavailable state, purge caches.

### 2.3 Cart tab screen
- [ ] Line list: image, name, variant chips, quantity stepper (PATCH with itemId — **cart item id, not product id**), swipe/button remove, line total.
- [ ] Price-drift indicator: when line `price` ≠ `product.priceAfterDiscount`, show the live price with "price updated" note (totals are already recomputed from live prices server-side).
- [ ] Stock-drift indicator: when line `quantity` > `product.quantity` or `product.status !== "ACTIVE"`, flag the line ("only N left" / "no longer available") and nudge fix/remove — this pre-empts checkout failures in Phase 5.
- [ ] Totals footer straight from `totalCartPrice` / `totalPriceAfterDiscount` (no client math), savings row when they differ.
- [ ] Clear-cart action with confirm dialog (204 handling per conventions).
- [ ] Empty state (`id: null` virtual cart) → CTA to Home.
- [ ] Remove-line 404 replay: the delete route is **not idempotent**; on 404 after a retry, treat as already-removed → refetch cart, no error toast.

### 2.4 Badge
- [ ] Tab-bar cart badge = sum of line quantities from the cart query cache (derived in a selector-style hook; no store duplication).

## Error handling matrix

| Code | Route | UX |
|---|---|---|
| `INSUFFICIENT_STOCK` | add / patch | inline per-line "only N available" from `errors[]` |
| `INVALID_VARIANT` | add | refresh product, reselect variant |
| `RESOURCE_NOT_FOUND` | add (product) | product unavailable state |
| `RESOURCE_NOT_FOUND` | patch/delete (item) | silent refetch (line already gone / cart identity changed) |
| `VALIDATION_ERROR` | any | guard client-side; log as bug if it fires |

## Definition of Done
- Cold start → add item → **kill and relaunch app** → cart persists (proves SecureStore lifecycle).
- First-add response token captured exactly once; second add carries the header (verify via dev proxy/logging **without** logging the token value).
- Quantity edit, remove, clear, and the not-idempotent-delete replay all behave per matrix.
- Cart totals match server strings verbatim after every mutation.
- Badge stays consistent across tabs without an app restart.

## Out of scope
Auth + merge (Phase 3), coupons (Phase 5).
