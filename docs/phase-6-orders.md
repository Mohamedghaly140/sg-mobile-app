# Phase 6 — Orders (History, Detail, Guest Tracking, Claim, Cancel)

**Objective:** the post-purchase surface: order history and detail for users, public guest tracking via the emailed token deep link, one-shot claiming of guest orders into an account, and self-cancel with correct state guards.

**Prerequisites:** Phase 5 DoD (orders exist to display).

**API surface:** `GET /orders`, `GET /orders/:id`, `GET /orders/guest/:token`, `POST /orders/claim`, `POST /orders/:id/cancel`.

## Tasks

### 6.1 Order history
- [ ] `useOrdersInfinite({status?})` — summary shape (`itemsCount` = distinct **lines**, not quantity sum; label it "items" carefully or show "N products"). Optional `OrderStatus` filter chips.
- [ ] List rows: `humanOrderId`, status pill (one color/token per `OrderStatus`), total, date. Newest first (server order).
- [ ] Refresh on screen focus + pull-to-refresh. **No polling loops** — explicit API rule.

### 6.2 Order detail
- [ ] `useOrder(id)` → full detail: line items with permanent price snapshots (copy note: prices shown are as-purchased), address-free totals block (`itemsSubtotal`, `discountApplied`, `shippingFees`, `totalOrderPrice` — displayed verbatim), status, `isPaid`, payment method.
- [ ] 404 covers both "missing" and "someone else's" by design → generic not-found state, invalidate list.

### 6.3 Guest tracking (public deep link)
- [ ] Route `orders/track/[token]` — **Public**, no auth gate. Renders the full detail shape from `GET /orders/guest/:token`.
- [ ] Deep linking: configure the app scheme + (if the storefront web domain hosts universal/app links later, note as follow-up) so the email link opens this screen with the token param. Token is throttled 10/60s → retry button, no auto-retry.
- [ ] `404 CLAIM_TOKEN_INVALID` → single message per API doc: "This tracking link is invalid or has expired." Do not attempt to distinguish expired/consumed/unknown.
- [ ] Never log the token; never echo it into analytics/navigation breadcrumbs.

### 6.4 Claim flow
- [ ] On the tracking screen: signed-out → "Create an account to keep this order" → Clerk sign-up/sign-in → return with token intact (pass through navigation state, not URL history where avoidable).
- [ ] Signed-in → `POST /orders/claim { token }` (validate 64-char length client-side first → avoids a pointless `VALIDATION_ERROR`).
- [ ] Success (200): navigate to the owned `orders/[id]`, invalidate `orderKeys.list`. Token is consumed — the tracking URL is now dead; make the redirect replace history.
- [ ] **Timeout-retry rule from the API doc:** if a claim request times out and the retry returns `404 CLAIM_TOKEN_INVALID`, treat it as "already claimed" → refetch order list, find the order, navigate. Encode this in the mutation's error handler with a "did we just time out?" flag.
- [ ] Throttle 5/60s → button disabled in flight.

### 6.5 Self-cancel
- [ ] Cancel button on order detail rendered **only when** `status === "PENDING" && !isPaid`; confirm dialog states stock returns and coupon use is released.
- [ ] `POST /orders/:id/cancel` success → `setQueryData(orderKeys.detail(id), updated)` + invalidate list.
- [ ] `409 INVALID_STATUS_TRANSITION` (admin moved it between render and tap — explicitly anticipated by the API doc) → refetch the order, show its current status, remove the button.

### 6.6 Post-checkout handoff polish
- [ ] Phase 5 confirmation screens now link: registered → `orders/[id]`; guest → explain the email tracking link (and, if the app is installed when tapped, it deep-links back here).

## Error handling matrix

| Code | Where | UX |
|---|---|---|
| `CLAIM_TOKEN_INVALID` | track / claim | single invalid-or-expired message; claim-after-timeout special case → treat as claimed |
| `INVALID_STATUS_TRANSITION` | cancel | refetch, show current status |
| `RESOURCE_NOT_FOUND` | detail / cancel | generic not-found, invalidate list |
| `RATE_LIMITED` | track / claim | wait message, manual retry |

## Definition of Done
- Full loop on a clean device: guest checkout → email link → tracking screen → sign up → claim → order in history → detail → (seed a PENDING unpaid order) cancel succeeds; a PROCESSING order shows no cancel button and a forced cancel returns the handled 409.
- Consumed token's tracking link shows the invalid/expired message.
- Status filter, pagination, and focus-refresh verified; no interval timers exist anywhere in the orders feature.

## Out of scope
Refund UX beyond displaying `REFUNDED` status; notifications (backend Phase 9).
