# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — invoice creation/payment concurrency proofs and prior financial/dine-in hardening are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — durable dine-in recovery and prior FX/CI work are merged and verified.
- Business Portal main: `76e39eb6ba937082684cc72189c34f7157b967a8` — Studio V2 palette insertion is merged with exact-head CI green. Historical Studio Wave A/C preview work remains reopened pending acceptance verification.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency and financial API/settings boundary work.

## Studio acceptance audit — current truth

Historical Wave A/B/C completion is not trusted merely because PR #86 had a green CI run. Each acceptance list is checked against current code and matching executable evidence before any wave is marked complete.

| Wave | Status | Code-verified evidence | Still required |
|---|---|---|---|
| A — Token foundation | **REOPENED** | PR #87 produced a tokenized preview implementation and a strict acceptance contract; the failing `1px` literal was correctly fixed and PR #87 exact-head CI then passed. PR #87 was closed because its diff exceeded the per-PR budget. | Reimplement preview tokenization in budgeted PR(s), ground every renderer token against the corresponding Flutter widget source, then pass exact-head CI and criterion-by-criterion acceptance. |
| B — Pointer drag with magnetic snap | **REOPENED / partial** | PR #88 (`fix(studio): port palette insertion to pointer events`) merged as `76e39eb6...`; exact-head run #253 passed smoke, tests and build. Pointer Events, capture, pointermove/up/cancel cleanup, before/after hit testing and click suppression are now on main. | Existing Wave B magnetic snap/fuse/settle acceptance still needs to be rechecked against the current Studio V2 path with executable evidence; do not call the wave complete yet. |
| C — Real device emulation | **REOPENED / partial** | Historical device tokens and responsive stage remain on main. PR #87 demonstrated a bounded `overflowY:auto` frame and dedicated scroll-content structure, but those changes were not merged. | Implement real bounded frame scrolling on a budgeted PR, then add executable proof for scrolling, responsive relayout and demonstrable clipping/overflow. |

## Active Studio implementation

- PR #87 is **closed/superseded**, not merged. Its exact-head fix commit was `e62b6c7ae99ca8771ad3ee1e177d4321f3c1ec73`; its exact-head CI run #252 passed, but the branch remains unsuitable for merge because its diff exceeded the per-PR change budget.
- PR #88 is **merged** at `76e39eb6ba937082684cc72189c34f7157b967a8`; this is now the authoritative pointer-insertion baseline.
- Next Studio work should be split into <=500-change PRs with at least one test each. The Wave A preview rewrite must be rebuilt surgically rather than replaying the oversized #87 diff.

## Important contracts

- Dine-in authority remains backend-owned: `OPEN → FINALIZED → CLOSED`.
- Business invoice creation and payment use deterministic durable idempotency anchors with replay rather than duplicate money movement.
- Studio remains semantic-tree based with no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Before any Studio wave is written as **complete**, inspect the acceptance list, inspect the current code, execute or cite the matching tests/CI, and record any residual gap.

## Residual risks / hygiene

- Wave A token values introduced in the superseded PR #87 are not accepted as fully grounded; current work must derive them from the Flutter widget implementations.
- Wave B's new palette interaction is merged, but the historical magnetic snap/fuse/settle acceptance is not yet re-proven on current Studio V2 code.
- Wave C scroll changes from PR #87 are not on main and therefore do not count as current implementation.
- Bundle-size warning (>500 kB chunk) remains a separate performance slice.
- Backend branch-deletion cleanup remains optional hygiene only where safe tooling supports it.
