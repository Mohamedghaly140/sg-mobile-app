# Phase 7 — Hardening & Release

**Objective:** production readiness: performance, resilience, accessibility, observability, store submission via EAS, and documented seams for the two known backend follow-ups (Geidea, notifications).

**Prerequisites:** Phases 0–6 DoD; design-system tokens delivered and merged into `theme/tokens.ts`.

## Tasks

### 7.1 Design-token integration pass
- [ ] Replace placeholder token values with the real design system; because no component holds raw literals (Phase 0 rule), this is a one-file change + visual QA sweep of every screen.
- [ ] Typography/spacing audit against the design spec; dark-mode decision recorded (if deferred, tokens stay single-scheme but the shape supports schemes).

### 7.2 Performance
- [ ] List audit: `FlatList` → `FlashList` where item counts warrant (product grid, orders); stable `keyExtractor`s (server ids), memoized cards, `getItemLayout`/`estimatedItemSize`.
- [ ] `expo-image` everywhere with caching + Cloudinary transform params for width-appropriate variants (thumbnail vs gallery).
- [ ] Re-render pass with React DevTools on the three hot screens (product list, cart, checkout review); fix with memo/selectors, not premature abstraction.
- [ ] Cold-start budget: measure; lazy-load checkout/orders route groups if needed.

### 7.3 Resilience & offline posture
- [ ] `onlineManager` wired to NetInfo; global offline banner; queries pause/resume correctly.
- [ ] Decision point (ADR): adopt `@tanstack/react-query-persist-client` for catalog cache persistence, or defer. Cart/orders are **never** persisted client-side beyond the session token — server truth only.
- [ ] Error boundary per route group rendering `ErrorState`; crash → boundary, not white screen.
- [ ] Slow-network QA (network conditioner): skeletons everywhere a spinner currently sits; no layout jumps on money strings.

### 7.4 Security & privacy sweep
- [ ] Grep-audit: no Clerk JWT or `sessionToken`/claim token in logs, analytics, Sentry breadcrumbs, or URLs (claim token appears in the deep-link URL by necessity — scrub it from any logging/reporting layer).
- [ ] SecureStore keys inventory documented; verify tokens cleared on the three lifecycle events (merge, guest checkout, clear).
- [ ] Dependency audit (`bunx expo-doctor`, `bun audit`).

### 7.5 Accessibility
- [ ] Labels/roles on interactive elements; hit-slop on steppers/hearts; dynamic type spot-check; contrast check once real tokens land; screen-reader pass on the checkout flow specifically.

### 7.6 Observability
- [ ] Sentry (or chosen tool) with release/dist tagging; `ApiError` enriched events (code + endpoint, **no tokens/PII**); TanStack Query error hooks report unexpected codes (`VALIDATION_ERROR` reaching production = client bug signal).

### 7.7 Release engineering
- [ ] `eas.json` profiles: development (dev client), preview (internal), production. Env per profile (`EXPO_PUBLIC_API_URL`).
- [ ] App icons/splash, `app.config.ts` metadata, versioning policy (runtimeVersion + `expo-updates` for OTA JS fixes).
- [ ] iOS: submit under the appropriate Apple Developer account; note team-account governance given prior Google Play account-termination history — **no third-party release-manager roles**; document who holds submission access.
- [ ] Android: Play Console setup on the correct account, internal testing track → closed → production.
- [ ] Store listing assets checklist; privacy manifest/data-safety forms reflecting Clerk + Sentry + Cloudinary image loading.
- [ ] Manual full-regression script: the union of every phase's DoD checklist, executed on one physical iOS and one physical Android device.

### 7.8 Forward seams (documented, not built)
- [ ] **Geidea (backend Phase 7):** flip `CARD.enabled` in the payment-method config; add the payment-session step after order creation per the future contract; checkout error resolver already routes `PAYMENT_METHOD_UNAVAILABLE`. Write the placeholder ADR referencing backend ADR-0002.
- [ ] **Notifications (backend Phase 9):** reserve `features/notifications`; push-token registration and an inbox screen when endpoints exist. Until then, focus-refresh remains the only status mechanism.

## Definition of Done
- Preview builds distributed and regression script passed on physical devices.
- No token/PII in any log or crash report (verified by inspection of real Sentry events).
- Production builds submitted to both stores; OTA update channel verified with a trivial JS change.
- The two seam ADRs merged.
