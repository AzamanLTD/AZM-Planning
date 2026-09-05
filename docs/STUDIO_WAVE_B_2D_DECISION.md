# Studio Wave B — 2D Surface Decision

**Date:** 2026-09-05 UTC
**Decision:** Do not restore the historical 2D canvas solely to satisfy the reopened magnetic-snap criteria.

## Evidence

Current `StorefrontStudioV2.jsx` is a semantic layer-tree editor. Its central stage mounts `StorefrontPhonePreview` inside the bounded device-frame scroll viewport. It does not mount `StorefrontCanvas.jsx`.

The legacy `StorefrontCanvas.jsx` remains a true 2D grid editor and owns the historical geometry engine (`gridGeometry`, `magneticSnap`, connected groups, pointer drag/resize and committed grid positions). Reintroducing it into V2 would therefore introduce a second interaction/layout authority unless the V2 document model first defines a deliberate 2D persistence contract.

## Decision

Keep Wave B at **partial / insertion-complete**:

- retain Pointer Events palette insertion, capture, cancellation and before/after semantic insertion in V2;
- keep the semantic document (`pages`, `nodes`, `children`, layout intent) as the current V2 persistence authority;
- do not port magnetic snap/fuse/settle into the V2 phone-preview surface by translating semantic nodes into a shadow grid;
- do not restore `StorefrontCanvas` merely to make historical acceptance boxes green.

## Trigger for reconsideration

A 2D surface becomes justified only when there is a product requirement for spatial placement that cannot be expressed by the semantic layout model. At that point, define first:

1. the canonical 2D coordinate model and responsive behavior;
2. the persistence/mutation boundary for positions and sizes;
3. interaction ownership between the renderer and editing overlay;
4. executable geometry tests for snapping, connected movement, resize, keyboard nudging and settle behavior.

Only after those contracts exist should the existing drag engine be adapted or reimplemented. This avoids carrying a legacy canvas and a semantic V2 canvas in parallel.
