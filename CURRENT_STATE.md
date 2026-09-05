# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — invoice creation/payment concurrency proofs and prior financial/dine-in hardening are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — durable dine-in recovery and prior FX/CI work are merged and verified.
- Business Portal main: `a1002e0846f7160298dc58e3f8261e675a9a1c5f` — PR #88 pointer insertion, PR #89 first Wave A renderer-token slice, and PR #91 second Flutter-grounded token slice are merged and verified. PR #91 exact-head CI run #258 passed smoke, tests and build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency and financial API/settings boundary work.

## Studio acceptance audit — current truth

Historical Wave A/B/C completion is not trusted merely because PR #86 had a green CI run. Each acceptance list is checked against current code and matching executable evidence before any wave is marked complete.

| Wave | Status | Code-verified evidence | Still required |
|---|---|---|---|
| A — Token foundation | **REOPENED / partial** | PR #89 merged a budgeted tokenized slice covering HeroHeader, QuickInfoBar, ProductGrid, ReviewCarousel and ContactCard. PR #91 adds Flutter-grounded token maps for Showcase Gallery, Location Map, and Video Player with executable token/scaling assertions. | Wire the newly grounded Showcase/Location/Video values into their renderers, tokenize the remaining renderer set and frame/chrome geometry, remove all remaining numeric inline pixel literals from preview style objects, ground every remaining token against Flutter source, then pass a full-wave criterion audit with exact-head CI. |
| B — Pointer drag with magnetic snap | **REOPENED / partial** | PR #88 (`fix(studio): port palette insertion to pointer events`) merged as `76e39eb6...`; exact-head run #253 passed smoke, tests and build. Pointer Events, capture, pointermove/up/cancel cleanup, before/after hit testing and click suppression are on main. | Recheck the historical magnetic snap/fuse/settle acceptance against current Studio V2 code with executable evidence; do not call the wave complete yet. |
| C — Real device emulation | **REOPENED / partial** | Device emulation tokens/stage remain on main. PR #87 demonstrated a bounded `overflowY:auto` frame and scroll-content structure, but those changes were not merged. | Implement real bounded phone-frame overflow on current main, then add executable proof for scrolling, responsive relayout, and demonstrable clipping/overflow. |

## Active Studio implementation

- PR #87 is **closed/superseded**, not merged. Its exact-head fix commit was `e62b6c7ae99ca8771ad3ee1e177d4321f3c1ec73`; exact-head run #252 passed, but the PR exceeded the normal per-PR budget and remains reference material only.
- PR #88 is merged at `76e39eb6ba937082684cc72189c34f7157b967a8`; this is the authoritative pointer-insertion baseline.
- PR #89 is merged at `59d9567d72b4eb0798a1a97f2fc9877381725a26`; this is the authoritative first Wave A renderer-token slice.
- PR #91 is merged at `a1002e0846f7160298dc58e3f8261e675a9a1c5f`; exact head `df1bcd0d2f6903d396bc95affec9311b306fbfa9` and run #258 passed smoke, tests and build. This is the authoritative second Wave A token-grounding slice for Showcase Gallery, Location Map, and Video Player.
- Next Studio work must remain <=500 changed lines per PR with at least one test. The remaining Wave A work is to be split rather than replaying #87.

## Important contracts

- Dine-in authority remains backend-owned: `OPEN → FINALIZED → CLOSED`.
- Business invoice creation and payment use deterministic durable idempotency anchors with replay rather than duplicate money movement.
- Studio remains semantic-tree based with no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Before any Studio wave is written as **complete**, inspect the acceptance list, inspect current code, execute or cite matching tests/CI, and record every residual gap.

## Residual risks / hygiene

- Wave A still has remaining renderers and shared frame/chrome geometry with numeric inline pixels on current main.
- Wave A token values not directly grounded in the current Flutter widget source remain unacceptable as completion evidence.
- Wave B's palette interaction is merged, but the historical snap/fuse/settle acceptance is not yet re-proven on current Studio V2 code.
- Wave C scroll changes from PR #87 are not on main and therefore do not count as current implementation.
- Bundle-size warning (>500 kB chunk) remains a separate performance slice.
- Backend branch-deletion cleanup remains optional hygiene only where safe tooling supports it.
