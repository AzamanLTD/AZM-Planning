# AZAMAN — Experience Blueprint + Marketplace Continuation — 2026-09-01

## Purpose

This document records the verified continuation state after the Experience Blueprint and marketplace-experience work converged. It is an engineering continuation point, not a completion declaration.

## Product architecture

The target architecture remains:

`Business Portal / Experience Studio → Backend-authoritative Experience Blueprint → Flutter marketplace renderer → category-native customer journey`

The Blueprint controls presentation/interaction grammar. Backend remains authoritative for availability, inventory, prices, bookings, orders, payments and financial state. Flutter remains the customer-facing renderer.

## Verified merged slices

### Backend

- PR #84 — Experience Studio draft source of truth: experience editor state moved into the storefront draft; public rendering reads the published storefront snapshot.
- PR #85 — template/revert lifecycle operations preserve the existing Experience Blueprint when legacy snapshots omit it.
- PR #86 — authenticated Experience Studio now exposes authoritative `navigationModes` and `detailPresentations` catalogs.
- PR #86 merged to Backend main as `ced1917845846bf0c31221c40ff81e5907626bac` after the full Azaman Test Suite passed, including PostgreSQL schema application, Prisma generation, Jest, and database backup/restore.

### Flutter

- PR #49 — hotel marketplace building/floor explorer.
- PR #50 — restaurant flip-book edge/spine-anchored interaction.
- PR #51 — canonical transit `BusSeatSelector` replaces the legacy booking seat grid.
- PR #52 — published Experience Blueprint fetched and supplied to the marketplace experience stage.
- PR #53 — typed Blueprint adapter makes published preset, navigation context, detail labels, commit affordance, motion tempo and reduced-motion behavior authoritative at the marketplace boundary.

### Business Portal

- PR #25 — generic storefront editor saves preserve the current Experience Blueprint snapshot.
- PR #26 — Experience Studio exposes navigation mode, detail presentation, detail controls and persistent tray/bag configuration, with backend catalogs when available and safe fallback catalogs.

## Backend invariant still being closed

The underlying legacy `storefrontService.saveDraft()` still accepts a layout object that omits `experience` and can therefore replace a draft snapshot without the Blueprint. The portal and Flutter clients defend against this, but the backend API boundary must not trust either client.

PR #87 is the active implementation:

`fix: enforce Experience Blueprint preservation at draft API boundary`

Head: `d7e350c3065f39b9c66716d46192af8ad24eca77`

The patch adds a small reusable `storefrontDraftExperienceGuard` and places an API-boundary `/me/draft` guard in the Experience Blueprint router, which is mounted before the generic storefront router. It preserves an existing `experience` only when the incoming layout omits the field; explicit incoming values, including `null`, remain authoritative. Focused tests cover omission, explicit replacement, null and no-existing-snapshot cases.

**Important limitation:** this closes the public API boundary but does not yet make the lower-level `storefrontService.saveDraft()` itself omission-safe for arbitrary future internal callers. That deeper service invariant remains a follow-up hardening target and should not be claimed complete from PR #87 alone.

PR #87 CI must be checked against the exact head before merge.

## Marketplace product direction

The four primary category-native journeys remain:

- Restaurant: menu exploration → dish focus/customization → order tray → checkout/order state.
- Retail: shelf/collection exploration → product quick look → variants/quantity → shopping bag → checkout.
- Hotel: building/floor traversal → room focus → gallery/specifications/availability → booking.
- Transit: trip identity → journey/vehicle context → physical seat selection → passenger details → booking/boarding.

Existing primitives are not considered finished merely because they exist. Product review must continue to test whether each interaction is useful, category-native, fast, accessible, and configurable rather than decorative.

## Current Flutter fidelity gap

The typed Blueprint adapter currently exposes policy metadata and commit affordances, but several fields still influence labels/context more than underlying primitive behavior:

- `navigation.mode` is primarily surfaced as navigation context; the primitives do not yet expose all four navigation modes as distinct customer navigation mechanisms.
- `detail.presentation` and its gallery/specification/options/quantity flags are not yet consistently wired into restaurant, retail, hotel and transit detail primitives.
- `persistentTray` changes some copy/affordance choices, but the marketplace business-profile flow does not yet provide a fully category-native persistent tray/bag/selection surface everywhere.

These are deliberate next product-engineering targets, not reasons to add arbitrary new remote-widget infrastructure.

## Next engineering loop

1. Finish and verify PR #87 with the exact Backend CI head.
2. Audit the remaining lower-level `saveDraft()` callers and move the preservation invariant into the service itself when the repository editing path allows a clean, reviewable change; keep the API-boundary guard until that deeper invariant is proven.
3. Trace the complete restaurant order path. The current marketplace restaurant dish callback opens the existing ticket/order sheet; the canonical cart provider already supports persistent per-business carts. Determine whether the correct convergence is to add dishes to the existing cart/tray without creating a competing order system.
4. Wire Blueprint detail policy into the real domain primitives rather than only labels. Use existing product `variants`/`modifierGroups`, room data, and transit seat state; do not invent fields.
5. Improve hotel room focus and transit journey/seat presentation where the existing primitives can support it without duplicating booking logic.
6. Continue the screen-level product audit for loading, empty, failure, retry, reduced motion, accessibility, long names, small devices and repeated entry/exit.
7. Update this planning record after each substantial merged slice with exact PR/commit/CI evidence.

## Non-goals

- No new Prisma models for presentation-only concepts unless existing JSON contracts genuinely cannot represent them.
- No second motion system.
- No second cart/order system when the existing cart provider/backend checkout can be extended safely.
- No frontend-only business configuration that the backend cannot validate and persist.
- No declaration of marketplace completion based solely on green CI.
