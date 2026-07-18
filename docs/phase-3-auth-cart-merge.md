# Phase 3 — Auth (Clerk), Cart Merge, Profile

**Objective:** sign-up/sign-in via Clerk Expo, automatic Bearer attachment, the anonymous→user cart merge executed correctly on sign-in, sign-out hygiene, and the profile screen.

**Prerequisites:** Phase 2 DoD (a guest cart must exist to prove the merge).

**API surface:** all **Auth**-mode plumbing from conventions; `GET /users/me`, `PATCH /users/me`. Merge uses `GET /cart` (no merge endpoint exists).

## Tasks

### 3.1 Clerk integration
- [ ] `(auth)` route group: sign-in, sign-up, verification screens using `@clerk/clerk-expo` hooks; email + password per current Clerk instance config (SMS OTP for Egyptian numbers is a known Clerk limitation — email-first UX).
- [ ] Wire the real `getToken` provider into the axios request interceptor (Phase 0 left it injectable). Confirm: fresh token per request, no caching.
- [ ] `<RequireAuth>` gate component: signed-out users on Auth screens (Account tab content, wishlist, addresses, orders) see an inline sign-in prompt; browsing tabs never hard-redirect.

### 3.2 Cart merge (the critical sequence)
Implement as one `onSignedIn` effect (fires when Clerk session becomes active):

1. If `cartSession.token` exists → `await queryClient.fetchQuery(cartKeys.current)` — the interceptor sends **both** the Bearer token and `X-Cart-Session`, which triggers the server-side merge and returns the merged user cart.
2. Then `cartSession.clear()` (API doc: mobile deletes the stored token after that authenticated response).
3. `setQueryData(cartKeys.current, mergedCart)` (fetchQuery already caches it) and invalidate nothing else.
4. If no guest token → just invalidate `cartKeys.current` so the user cart loads.

- [ ] Merge is replay-safe server-side (merged anonymous cart ceases to exist), but still guard the effect against double-fire (ref flag) to avoid redundant requests.
- [ ] Note the documented adoption nuance: when the user had no cart, adopted-cart totals may be stale until the next mutation — display as returned; no client correction.
- [ ] **Registered checkout dependency (Phase 5):** checkout loads the cart by user ID only, so this merge-on-sign-in guarantees the user never checks out a partial cart. Document the invariant in code comments.

### 3.3 Sign-out hygiene
- [ ] On sign-out: Clerk `signOut()` → clear the **entire** query cache (`queryClient.clear()`) → cart badge recomputes from the now-guest `GET /cart` (virtual empty or a pre-existing… none: guest token was consumed at merge, so signed-out users start clean).

### 3.4 Account disabled
- [ ] Global handler (query client error hook): `403 ACCOUNT_DISABLED` → force Clerk sign-out → full-screen "account disabled" state with support contact. One implementation, all features inherit it.

### 3.5 Profile feature
- [ ] `useMe()` → Account tab header (composed `name`, `email`, `phone`, member-since).
- [ ] Edit screen: initialize `firstName`/`lastName` **from Clerk's user object**, never by splitting the backend composed `name` (explicit API rule). Phone field with Egyptian format validation client-side.
- [ ] `PATCH /users/me` rules encoded in the form: name fields submitted as an atomic pair (both or neither); phone-only updates omit both; never send a `name` field.
- [ ] `409 DUPLICATE_RESOURCE` → inline "phone already in use" on the phone field; `422 VALIDATION_ERROR` → map `errors[]` to fields.
- [ ] On success: invalidate `profileKeys.me`; reflect name change in Clerk-driven UI (Clerk user object refresh).

## Error handling matrix

| Code | Where | UX |
|---|---|---|
| `UNAUTHENTICATED` | any Auth route | re-auth prompt, retry after sign-in |
| `ACCOUNT_DISABLED` | anywhere | global sign-out + disabled screen |
| `DUPLICATE_RESOURCE` | profile phone | field-level error |
| `VALIDATION_ERROR` | profile | field mapping via `getValidationErrors` |

## Definition of Done
- Guest with 2 cart lines signs up → cart survives merge; relaunch shows the user cart; stored guest token is gone from SecureStore.
- Guest cart + **existing** user cart with one overlapping `(productId, color, size)` line → quantities summed (capped at stock) after sign-in; distinct lines appended. (Seed via a second device/session.)
- Sign out → cart empties, Auth screens gate, catalog still browsable.
- Profile edit: name pair, phone-only, duplicate phone, and single-name-field rejection all behave per matrix.
- No request ever logs or caches the Clerk JWT.

## Out of scope
Wishlist/addresses/orders screens (they gate behind `<RequireAuth>` but ship in Phases 4/6).
