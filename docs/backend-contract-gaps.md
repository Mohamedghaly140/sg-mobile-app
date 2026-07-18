# Backend Contract Gaps — Questions & Requests

> **Audience:** backend team (`sg-couture-api`).
> **Source:** mobile dev-plan review against `docs/integration/storefront/` (contract snapshot dated 2026-07-18).
> **Status:** open — each item lists what blocks us, what we need, and our fallback if the contract stays as-is.
>
> Resolutions should land as updates to the integration guide (and `API_SPECIFICATION.md` where behavior changes), then this file gets updated/pruned.

---

## GAP-1 · Expired guest cart: is a new `sessionToken` minted?

**Blocks:** Phase 2 (guest cart). **Severity: high** — silent cart loss if we guess wrong.

**Contract today:** `00-conventions.md` says the **first** anonymous `POST /cart/items` returns `sessionToken` once, and later responses omit it. Anonymous carts expire 7 days after the last mutation.

**The undefined case:** the app still holds a token whose cart has expired (or was deleted). The next `POST /cart/items` creates a cart lazily — but under which identity?

- If the server **reuses the presented `X-Cart-Session` token**, our stored token stays valid and nothing special is needed.
- If the server **mints a new token**, the response now contains a `sessionToken` that is *not* the first mutation of this app install. A client that only captures the token "once" misses it, and every subsequent call references a dead cart → the customer's cart silently vanishes on every add.

**What we need:**

1. A sentence in `05-cart.md` / `00-conventions.md` specifying the behavior when `X-Cart-Session` references a nonexistent/expired cart, for each of `GET /cart`, `POST /cart/items`, and the other mutations.
2. If a new token can be minted: confirmation that `sessionToken` appears in the response **whenever the acting token changes**, not only on the literal first-ever mutation.

**Our fallback (defensive, implemented regardless):** the app persists `sessionToken` from **any** cart response that contains one, overwriting the stored value — "capture once" becomes "capture whenever present". Please confirm this is safe and sufficient.

---

## GAP-2 · `REVIEW_EXISTS` recovery: no way to fetch the user's own review

**Blocks:** Phase 4 (review CRUD). **Severity: medium** — feature works but has a broken edge on popular products.

**Contract today:** `POST /products/:id/reviews` returns `409 REVIEW_EXISTS`, and `03-reviews.md` says to use the existing review's `id` with `PATCH /reviews/:id`. But there is no endpoint that returns the current user's review for a product. The only source is the **public paginated list** (`GET /products/:id/reviews`, 20/page, newest first) — on a product with many reviews, the user's review can sit pages deep.

**What we need (either one):**

- **Preferred:** a lightweight owned-review lookup, e.g. `GET /products/:id/reviews/me` → the caller's review or 404. Auth mode: Auth.
- **Alternative:** a `mine=true` filter on the existing list route, or include the caller's review `id` in the `409 REVIEW_EXISTS` error payload (a structured `errors: [{ reviewId }]` line would be enough).

**Our fallback:** on 409, page through the review list matching `user.id` against the Clerk user id. Works, but is throttle-unfriendly (100 req/60s global) and O(pages) on popular products; we'd rather not ship it.

---

## GAP-3 · Checkout preview subtotal: confirm `cart.totalPriceAfterDiscount ≡ itemsSubtotal`

**Blocks:** nothing hard — Phase 5 (checkout review step). **Severity: low** — terminology/invariant confirmation.

**Contract today:** the checkout review screen must show an items subtotal before any coupon is applied. `itemsSubtotal` exists only in the coupon-validate response ("cart total **after product discounts**, before the coupon") and in the order response. The cart shape exposes `totalPriceAfterDiscount` (recomputed from live prices on every mutation).

**What we need:** one sentence in `05-cart.md` or `06-coupons.md` confirming that for the same cart state, `cart.totalPriceAfterDiscount` equals coupon-validate/order `itemsSubtotal` — i.e. the app may render `totalPriceAfterDiscount` as the "Subtotal" line when no coupon has been previewed. If any case breaks the equality (rounding, scale, timing), document it.

**Our fallback:** display `totalPriceAfterDiscount` as the subtotal until a coupon preview/order response supplies `itemsSubtotal`, then prefer the server's value. No client arithmetic either way.

---

## GAP-4 · Order detail contains no shipping address

**Blocks:** nothing in the current plan — Phase 6 renders an "address-free" detail by design. **Severity: product decision needed.**

**Contract today:** the order detail shape (`09-checkout.md`, reused by `GET /orders/:id` and `GET /orders/guest/:token`) has items, totals, status, payment fields — but **no shipping address and no contact/notes**. Customers therefore can never see *where* an order is being delivered, and support conversations ("which address did I use?") have no in-app answer. Deleting a saved address also severs the order's address relation (`08-addresses.md`), so the data may genuinely be gone unless it's snapshotted.

**What we need:** a decision, then contract text:

1. Is the shipping destination snapshotted on the order server-side? (If not, that's the prerequisite.)
2. If yes: add a read-only `shippingAddress` (and ideally `notes`, and `contact` for guest orders) block to the detail shape, or explicitly document that the storefront intentionally never exposes it.

**Our fallback:** ship Phase 6 as planned (no address shown). UI copy avoids implying the address is viewable.

---

## Resolution log

| Gap | Status | Resolved by / notes |
|---|---|---|
| GAP-1 | open | — |
| GAP-2 | open | — |
| GAP-3 | open | — |
| GAP-4 | open | — |
