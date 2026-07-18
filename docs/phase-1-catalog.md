# Phase 1 — Catalog (Products, Categories, Public Reviews)

**Objective:** a guest can browse the entire catalog: featured home, category tree, filtered/sorted/paginated product lists, full product detail with gallery and public reviews. Everything here is **Public** auth mode.

**Prerequisites:** Phase 0 DoD.

**API surface:** `GET /products`, `GET /products/:slug`, `GET /categories`, `GET /categories/:slug`, `GET /products/:id/reviews`.

## Tasks

### 1.1 Categories feature
- [ ] `features/categories`: `api.ts` (`getCategories`, `getCategory(slug)`), `keys.ts`, `hooks.ts` (`useCategories` — unpaginated, staleTime 5 min; counts are display hints per API doc).
- [ ] Categories tab screen: category list with image + `productCount`, expandable sub-categories. Tapping navigates to `products?category=<slug>` or `products?subCategory=<slug>` — always **slugs**, never IDs (API doc requirement).

### 1.2 Products feature — list
- [ ] `ProductFilters` type mirroring every documented query param (`search`, `category`, `subCategory`, `minPrice`, `maxPrice`, `sizes[]`, `colors[]`, `featured`, `sort`). Serializer converts to query params (`sizes` → CSV) — one function, unit-tested.
- [ ] `useProductsInfinite(filters)` via `useInfiniteQuery`; `getNextPageParam` from `meta.hasNext`/`meta.page`.
- [ ] Product list screen: `FlatList` 2-column grid of `ProductCard` (image, name, price + strikethrough original when `discount` > 0, rating, low-stock badge when `quantity` small, sold-out badge at 0 — **hint copy only**, card stays tappable per "advisory UX data").
- [ ] Filter sheet (bottom sheet): sort enum, price range, sizes/colors multi-select, featured toggle. Draft lives in `stores/ui.ts`; applying updates route params so lists are shareable/restorable.
- [ ] Search input on Home/list header (debounced 300 ms, ≤ 100 chars trimmed to match validation).
- [ ] Empty results → `EmptyState` with "clear filters" action.

### 1.3 Products feature — detail
- [ ] `useProduct(slug)`; detail screen: swipeable gallery (`images` ordered by `sortOrder`, tolerate null `imageUrl` entries), name, price block, description, category/sub-category chips (navigate to filtered list), size/color **selectors** (selection state local — becomes the add-to-cart variant in Phase 2), rating summary.
- [ ] `ratingsAverage: null` renders "No reviews yet", not 0 stars.
- [ ] 404 `RESOURCE_NOT_FOUND` → dedicated "product unavailable" state + remove any cached detail/list entry for that slug (API doc: purge stale cards).

### 1.4 Reviews (read-only)
- [ ] `features/reviews`: `useProductReviewsInfinite(productId)`; paginated section on product detail (first page inline, "see all" pushes a full list screen). Note the quirk: unknown product ID returns an empty page, not 404 — render as "no reviews".

### 1.5 Home tab
- [ ] Featured rail (`featured=true`, limit 10), newest rail (`sort=newest`), category shortcuts. All reuse the products hooks — no bespoke endpoints.

## Error handling matrix

| Code | Where | UX |
|---|---|---|
| `RESOURCE_NOT_FOUND` | product detail, category detail | unavailable state; purge cache entry |
| `VALIDATION_ERROR` | list filters | should be unreachable if serializer is correct → treat as bug, log |
| network / 5xx | all | `ErrorState` with retry; infinite lists keep loaded pages |

## Definition of Done
- Browse: home → category → filtered list → detail → reviews, on both platforms, as a guest.
- Filters combine correctly (verify AND semantics + any-match for sizes/colors against live data).
- Infinite scroll paginates and preserves scroll position on back-navigation.
- Prices with and without trailing zeros render identically formatted.
- `top_rated` sort shows unrated products last (server behavior — just verify display).

## Out of scope
Add to cart (Phase 2), wishlist buttons (Phase 4 — render a disabled placeholder now), review writing (Phase 4).
