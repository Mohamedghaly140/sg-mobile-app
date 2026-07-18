# Phase 4 — Account Features (Wishlist, Review CRUD, Addresses)

**Objective:** the signed-in experience beyond the cart: wishlist with optimistic UI, writing/editing/deleting own reviews, and the saved-address book that registered checkout (Phase 5) requires.

**Prerequisites:** Phase 3 DoD.

**API surface:** `GET/PUT/DELETE /wishlist(/:productId)`, `POST /products/:id/reviews`, `PATCH/DELETE /reviews/:id`, all six `/addresses` endpoints.

## Tasks

### 4.1 Wishlist
- [ ] `useWishlist()` (unpaginated array, newest first), `useToggleWishlist(productId)`.
- [ ] **Optimistic updates with rollback** — explicitly sanctioned here because both PUT (returns `{added:true}` even when already present) and DELETE (204 even when absent) are idempotent. `onMutate` snapshot → rollback `onError`; `onSettled` invalidate.
- [ ] Heart button on `ProductCard` + product detail (replaces Phase 1 placeholder). Signed-out tap → sign-in prompt (wishlist is Auth-only; anonymous wishlist is out of scope by API design).
- [ ] Wishlist screen (Account tab entry): product cards + `addedAt`; `available: false` items render disabled with "no longer available", **not linked** to product detail (API rule). Remember: ACTIVE-but-zero-stock is still `available: true` — show sold-out hint instead.
- [ ] `PUT` 404 (product id gone entirely) → rollback + toast.

### 4.2 Review CRUD
- [ ] "Write a review" on product detail for signed-in users: title (≤150, optional) + rating input constrained to 0.5 steps in 1–5 (send as **number**; comes back as string — types already reflect this from Phase 1).
- [ ] `409 REVIEW_EXISTS` → switch UI to edit mode with the user's existing review (find it in the review list by user id; PATCH with its id — retrying POST can never succeed, per API doc).
- [ ] Edit + delete own review (delete = 204, confirm dialog). Owner check client-side by comparing `review.user.id` with Clerk user id; server enforces `FORBIDDEN` regardless.
- [ ] All three mutations invalidate `reviewKeys(productId)` + `productKeys.detail(slug)` (aggregates recomputed transactionally server-side; last-review deletion yields `ratingsAverage: null` — Phase 1 already renders that).
- [ ] Product must be ACTIVE to review; 404 on POST → product-unavailable handling.

### 4.3 Addresses
- [ ] `features/addresses`: list (default first, then newest — server ordering, don't re-sort), create, edit, delete, set-default.
- [ ] Address form implementing the full DTO: required `alias/country/governorate/city/area/phone/addressLine1/details`, optional `postalCode/latitude/longitude`. Egyptian phone validation client-side mirroring the server; `VALIDATION_ERROR` field mapping as the backstop.
- [ ] Country/governorate/city entered with the **same canonical labels** the shipping-zone config uses (free text v1, but centralize the label constants — shipping resolution is not case-normalized, per API doc; Phase 5 depends on this).
- [ ] Default handling: set-default uses `PATCH /addresses/:id/default` exclusively — **no free-form `isDefault` checkbox on the edit form** (API doc explicitly warns the literal `isDefault:false` path can leave no default).
- [ ] Delete: confirm dialog; server auto-promotes a new default — invalidate list, no client-side promotion logic.
- [ ] First-address nuance: server forces default; UI just reflects the response.
- [ ] All 404s (missing **or other user's** address — indistinguishable by design) → refetch list + "address not found".

## Definition of Done
- Wishlist heart flips instantly offline-fast, rolls back on failure, and reconciles on refetch; DRAFT/ARCHIVED items render disabled.
- Second review attempt lands in edit mode, not an error dead-end.
- Rating aggregates on the product card/detail update after create/edit/delete without app restart.
- Address CRUD + default promotion verified, including deleting the default.
- Every form maps `errors[]` field paths to inputs.

## Out of scope
Using addresses at checkout (Phase 5).
