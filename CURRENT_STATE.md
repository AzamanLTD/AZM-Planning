# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — invoice creation/payment concurrency proofs and prior financial/dine-in hardening are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — durable dine-in recovery and prior FX/CI work are merged and verified.
- Business Portal main: `62ceb099cafd1832cb45ba7cb14f14e18d4c2c1f`. Historical Studio Wave A/B/C implementation is merged, but wave completion is **reopened** pending acceptance verification.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency and financial API/settings boundary work.

## Studio acceptance audit — current truth

Historical Wave A/B/C completion is not trusted merely because PR #86 had a green CI run. The acceptance lists are now checked against current code before any wave is marked complete.

| Wave | Status | Code-verified evidence | Still required |
|---|---|---|---|
| A — Token foundation | **REOPENED** | Preview renderer geometry is being routed through `toPreviewPx()` and shared token data on PR #87; 16-widget registry coverage is preserved. | Exact-head CI and final criterion audit, including token grounding against corresponding Flutter widgets. |
| B — Pointer drag with magnetic snap | **REOPENED / partial** | Studio V2 palette insertion is implemented with Pointer Events, pointer capture, pointermove/up/cancel, and before/after target hit testing; HTML5 palette `draggable` path removed. | Full capture/snap/fuse/settle executable tests and complete no-HTML5-drag acceptance. |
| C — Real device emulation | **REOPENED / partial** | Phone frame now has bounded tokenized height, `overflowY: 'auto'`, and explicit scroll-content structure. | Executable proof of real scrolling plus visible responsive relayout and demonstrable clipping/overflow. |

## Active Studio implementation

Business Portal PR #87 (`fix(studio): reopen Wave A/C acceptance and port palette pointer drag`) is open and must not be treated as merged or complete until exact-head CI passes and the acceptance audit above is rerun.

## Important contracts

- Dine-in authority remains backend-owned: `OPEN → FINALIZED → CLOSED`.
- Business invoice creation and payment use deterministic durable idempotency anchors with replay rather than duplicate money movement.
- Studio remains semantic-tree based with no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Before any Studio wave is written as **complete**, inspect the acceptance list, inspect the current code, execute or cite the matching tests/CI, and record any residual gap.

## Residual risks / hygiene

- PR #87 is intentionally not merged yet because its diff currently exceeds the normal per-PR budget and its exact-head CI has not produced evidence yet.
- Newly introduced renderer token values still need measurement grounding against the Flutter widget implementations.
- Bundle-size warning (>500 kB chunk) remains a separate performance slice.
- Backend branch-deletion cleanup remains optional hygiene only where safe tooling supports it.
