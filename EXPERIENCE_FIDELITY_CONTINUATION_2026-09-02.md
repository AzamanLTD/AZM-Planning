# AZAMAN — Experience Fidelity Continuation — 2026-09-02

## Verified state

The Experience Blueprint hardening slice has now reached the backend API boundary and the Business Portal preview layer.

### Backend

- PR #86 merged as `ced1917845846bf0c31221c40ff81e5907626bac`.
- PR #87 merged as `65f370cde2c216f9cf1ad5883482fab9f8cdef8b`.
- The authenticated Experience Studio endpoint now exposes authoritative navigation-mode/detail-presentation catalogs.
- The generic `/me/draft` API boundary preserves an existing Experience Blueprint when the incoming layout omits `experience`; explicit values, including `null`, remain authoritative.
- Full PostgreSQL-backed Azaman Test Suite passed for the merged backend slices.

## Business Portal

- PR #33 is merged as `95d3ddee3005ea8a071788acac8a2a41977311bb`.
- Experience Studio now previews the actual category-native interaction grammar rather than presenting a static configuration checklist.
- The simulator covers dining, retail, hotel and transit journeys, focused detail states, and configured commit metaphors.
- CI run #84 passed smoke tests, 45 Vitest tests, and production build after correcting stale stage assertions.

The simulator remains a fidelity preview. It does not own live inventory, pricing, authorization, payment or fulfillment.

## Product principle

The platform should not feel like four collections of themed widgets. It should feel like four different kinds of experiences.

- Restaurant: browse as a menu, focus a dish, configure meaningful options, add to a persistent tray, and continue ordering without losing context.
- Retail: move through collections/shelves, pull a product forward, inspect variants, quantity and availability, then lift it into a persistent bag.
- Hotel: traverse the property/floors, focus a room, inspect imagery/specifications/availability, and transition naturally into booking.
- Transit: establish trip context, move through the vehicle/journey, select a physical seat, attach passenger identity, and continue into booking/boarding.

Every animation must reinforce the physical metaphor rather than merely decorate it. Motion should have a clear cause, destination, duration, interruption behavior, reduced-motion fallback, and recovery path.

## Important fidelity finding

Flutter already contains an `edgeAnchored` FlipBook controller mode and tests proving that it restricts grabs to the outer edge and stabilizes the vertical curl axis. The older restaurant flip-book consumer already opts into this mode. The newer `RestaurantNativeMenuJourneyClean` experience currently constructs `FlipBook` without an edge-anchored controller, so the clean/native restaurant path can still exhibit the corner-curl behavior that the product direction explicitly rejects.

Next Flutter slice should wire the existing edge-anchored controller into the native restaurant journey rather than creating another flip-book implementation.

## Next product-engineering loop

1. Verify the post-merge Business Portal main CI and Backend main CI.
2. Wire the existing edge-anchored FlipBook controller into the native restaurant journey and add a focused widget/regression test.
3. Audit the restaurant detail surface against the real cart provider: dish options, sizes, quantity, table/dine-in context, add-to-tray animation, persistent tray state, retry and failure behavior.
4. Audit hotel room traversal/detail against the existing reservation/room contracts rather than introducing new presentation data.
5. Audit transit seat selection for physical-map clarity, selected/occupied/accessibility states, touch target behavior, zoom/pan interruption, and passenger association.
6. Continue wiring Blueprint detail/navigation policies into actual domain primitives; configuration must change behavior, not only labels.
7. Deep-audit all remaining marketplace screens for redundant sections, weak hierarchy, generic copy, unnecessary cards and decorative interactions. Remove or consolidate only after tracing references and proving the canonical path.
8. Update this planning record after each substantial merged slice with exact PR, commit and CI evidence.

## Non-goals

- No new Prisma models for presentation-only concepts.
- No second cart/order system.
- No second realtime source of truth.
- No ad-hoc workflow/code-rewrite automation.
- No category-specific hard-coded behavior that bypasses the Experience Blueprint contract.
- No completion claim based solely on green CI; product usefulness, motion quality, accessibility and failure behavior must also be reviewed.
