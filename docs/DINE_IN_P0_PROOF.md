# Dine-in P0 Proof

**Date:** 2026-09-05 UTC  
**Scope:** Flutter → Backend → Business Portal → Admin visibility

## Result

The dine-in financial/state path is implementation-verified across the active repositories. The remaining gap is not a known settlement defect: there is no single deployed four-surface E2E harness, and Admin Portal has no dedicated dine-in lifecycle projection identified in its current tree.

The correct completion boundary is therefore **cross-client implementation proof verified; deployed E2E/Admin projection residual remains explicit**.

## Evidence matrix

| Surface | Authority / contract | Executable evidence | Current result |
|---|---|---|---|
| Flutter | Payment request may fail ambiguously; client rereads the same tab and accepts success only from durable CLOSED state. Socket events are invalidation signals. | Frontend PR #90 + PR #91. PR #91 exact head `d1dbc94ed890583241a4d338fc2a045cf5cec4a3`, CI run `33967426867`, passed Analyze + Test + coverage upload. | **Verified** |
| Backend | `OPEN → FINALIZED → CLOSED`; invoice creation/payment is canonical; payment and tab closure share one transaction; deterministic `DINE_IN_TAB:<tabId>` idempotency; replay is non-charging. | Backend PR #233 exact head `2453729846cf15d95bf6ab637d26ae19f3b735fa`, CI run `33957019469`, passed production audit, full tests and DB recovery. Supporting PRs #232, #225, #234 and #235 are merged/verified. | **Verified** |
| Business Portal | Business owner receives persisted `DINE_IN_TAB_*` notification events; socket payloads only invalidate canonical query roots (`dine-in`, `dine-in-tabs`, `openTabs`, `dineInTab`). | `useBizNotifications.js` current main contract plus backend business-notification tests. | **Verified** |
| Admin | P0 roadmap expects visibility of high-value lifecycle state. | Current Admin tree contains financial/control-plane projections but no dedicated dine-in projection was identified. | **Residual / explicit gap** |

## Critical invariants proven

1. A finalized tab is settled through the canonical business invoice service; no second financial authority is introduced.
2. Tip is carried into the paid invoice and the closed tab total.
3. Payment and `FINALIZED → CLOSED` transition share a transactional boundary.
4. A committed `PAID/CLOSED` replay returns durable state without charging again.
5. Customer payment recovery never treats a socket event as financial proof.
6. Business notifications are owner-scoped and are convergence signals, not source-of-truth state.

## What is not claimed

This document does **not** claim a live production-like end-to-end run across four separately deployed processes. That evidence requires a runnable environment connecting the actual Flutter client, backend instance/database, Business Portal socket/query path and an Admin-visible projection.

## Next closure condition

Close the P0 residual only when either:

- a four-surface automated/staging harness exists and passes the finalize → payment → replay → reconnect/background scenarios; or
- Architecture explicitly removes Admin dine-in visibility from the release contract and records that decision, leaving the four-surface proof boundary to Flutter → Backend → Business Portal.
