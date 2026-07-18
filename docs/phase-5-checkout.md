# Phase 5 — Checkout (Shipping, Coupons, Registered + Guest)

**Objective:** convert carts into orders through both checkout paths, CASH-only, with the full documented failure matrix handled line-by-line. Checkout is atomic server-side; the app's job is a truthful preview, a single well-guarded submit, and precise recovery UX.

**Prerequisites:** Phase 4 DoD (addresses) for registered checkout; Phase 2 for guest checkout. Phases can overlap internally: 5.1–5.2 → 5.3 (guest) and 5.4 (registered) in either order → 5.5.

**API surface:** `GET /shipping/fee`, `POST /coupons/validate`, `POST /orders`, `POST /orders/guest`.

## Tasks

### 5.1 Shipping estimate
- [ ] `useShippingFee({country, governorate, city})` — enabled only when country+governorate present; keyed on the trio.
- [ ] Display fee + which zone won (null `zone.city` = governorate-wide rate). Copy marks it an **estimate**; the order response owns the final fee.
- [ ] `422 SHIPPING_NOT_AVAILABLE` → surface **at address selection time**, before the final step (API doc: don't let customers discover it at the last step). Block continue with "we don't deliver there yet".

### 5.2 Coupon preview
- [ ] Coupon field in checkout: normalize client-side (trim, uppercase, `^[A-Z0-9_-]{3,30}$`) before calling; guests include `email` once contact email is entered (per-user limit keyed on it).
- [ ] `useValidateCoupon` mutation (10/60s throttle → disable while pending, no auto-retry). Success renders `discountApplied` against `itemsSubtotal`; note coupons never discount shipping.
- [ ] Distinct copy per code: `RESOURCE_NOT_FOUND` (invalid), `COUPON_EXPIRED`, `COUPON_INACTIVE`, `COUPON_EXHAUSTED` (409), `COUPON_USER_LIMIT` (409).
- [ ] Preview is not a guarantee — keep the code in checkout state (`stores/ui.ts` checkout slice), re-validated implicitly by the checkout transaction; be ready for the same codes at submit.
- [ ] Signed-in + leftover guest header: this route is cart-aware and may trigger the merge — harmless because Phase 3 already cleared the token post-merge, but don't special-case it.

### 5.3 Guest checkout flow
Route group `checkout/guest`: contact → shipping address → review → placed.

- [ ] Contact form (`name`, Egyptian `phone`, `email` — email receives confirmation + claim link) and shipping form (same field set as addresses minus alias/default). Reuse Phase 4's address form component with a mode prop.
- [ ] Shipping fee preview auto-runs when country/governorate/city are filled (5.1).
- [ ] Review step: lines from live cart, `itemsSubtotal` + coupon preview + fee estimate, payment method selector (see 5.6), notes (≤1000).
- [ ] Submit `POST /orders/guest` — the anonymous identity header rides along automatically. Missing identity → `CART_EMPTY` (also covers a truly empty cart) → route back to cart tab.
- [ ] Success: order confirmation screen from the response detail shape; `claimToken: "sent-by-email"` is a **marker** — the real token only arrives by email. Copy: "We emailed your order tracking link to …". Then `cartSession.clear()` (API doc: drop the stored token after guest checkout) and `setQueryData(cartKeys.current, emptyCart)`.

### 5.4 Registered checkout flow
Route group `checkout`: address select → review → placed.

- [ ] Precondition (invariant from Phase 3): merge already happened at sign-in; still call `GET /cart` on checkout entry to render current lines/stock.
- [ ] Address selection from saved addresses (default preselected) + inline "add new" reusing Phase 4 form; fee preview per selected address.
- [ ] Submit `POST /orders` with `shippingAddressId`, `paymentMethod`, optional `couponCode`/`notes`.
- [ ] Success: confirmation from response (`humanOrderId` displayed; `id` used for navigation), cart cache reset, invalidate `orderKeys.list`.

### 5.5 Failure recovery (both flows)
Submit is throttled **5/60s** — `Button loading` disables while in flight; on `429 RATE_LIMITED`, keep **all** form/cart state and show a wait message (API doc mandates preserving state).

On failure, nothing was reserved or charged (full rollback). Handle:

| Code | Recovery UX |
|---|---|
| `INVALID_VARIANT` (422) | Map `errors[]` `{productId,color,size}` to rendered lines; refetch cart; prompt fix/remove per line |
| `INSUFFICIENT_STOCK` (409) | Map `errors[]` `{productId,requested,available}`; show available per line; refetch cart |
| `CART_EMPTY` | Route to cart tab with explanation |
| `SHIPPING_NOT_AVAILABLE` | Return to address step, mark destination unsupported |
| coupon codes (4×) | Return to review step, clear/flag coupon with the distinct copy from 5.2 |
| `RESOURCE_NOT_FOUND` (registered) | Address vanished (deleted on another device) or unknown coupon — refetch addresses / flag coupon |
| `PAYMENT_METHOD_UNAVAILABLE` | Should be unreachable (5.6); if it fires, fall back to CASH and log |

- [ ] Build one `CheckoutErrorResolver` mapping `ApiError` → `{step, lineAnnotations, message}` so both flows share recovery logic. Unit-test it against every code above with the documented JSON shapes.

### 5.6 Payment method abstraction (Geidea seam)
- [ ] `PaymentMethod` selector renders from a config list: `[{ id: 'CASH', enabled: true }, { id: 'CARD', enabled: false, badge: 'coming soon' }]`. CARD is visible-disabled, never submittable, so backend Phase 7 flips one flag + adds the Geidea session flow behind this seam. **Do not** call `POST /orders/:id/payment-session` — it does not exist yet.

## Definition of Done
- Guest: browse → cart → guest checkout → confirmation, on a device with no account. Confirmation email received with claim link (verifies backend wiring end-to-end).
- Registered: sign in → merged cart → address → coupon applied → order placed; totals on the confirmation match the server response strings verbatim (`totalOrderPrice = itemsSubtotal − discountApplied + shippingFees` — displayed, not computed).
- Forced-failure drills: deplete stock from another session mid-checkout → `INSUFFICIENT_STOCK` line recovery works; archive a product → `INVALID_VARIANT`; unsupported governorate → blocked at address step.
- Double-tap submit produces exactly one request; 429 path preserves the form.
- Cart is empty (cache + server) after either success; guest SecureStore token deleted.

## Out of scope
Order history/tracking/claim/cancel (Phase 6), CARD payment (backend Phase 7).
