# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main includes verified PRs #144–#155, #157–#162 | Cross-client dine-in lifecycle/realtime verification |
| `AZM-adminPortal` | Main lineage includes PR #95 and prior financial hardening | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44, #45 and branch-aware dine-in convergence PR #49 | Post-convergence lifecycle/realtime verification and UI state hardening |
| `AZM-frontend` | Main includes verified PRs #78 and #79 | Dine-in retry/realtime convergence |
| `AZM-Planning` | Navigation ledger synchronized through Backend #162, Business Portal #49 and Frontend #79 | Keep navigation synchronized after each verified merge |

## 2. Verified backend hardening now on main

- PR #131 payroll Serializable retry.
- PR #132 shift lifecycle-status mutation boundary.
- PR #135 dine-in settlement/finalization/payment-idempotency hardening.
- PR #136 KYB gate fail-closed hardening.
- PR #137 canonical Business OS Finance routing/runtime repair.
- PR #141 inventory restock atomicity (`a0876b2f61d5bc73acb1a1d76368e019d079fe82`).
- PR #143 dine-in test mock correction.
- PR #144 kiosk scoped capability + PIN rate limit (`22ee6db6633322bca8be3c60a346f974e5e9323c`).
- PR #145 POS settlement/inventory atomicity (`301a5795898f4b7de3b69c8156afe027f82e5155`).
- PR #146 duplicate-line recipe consumption (`2465c01b1746cec95cdec5adcd0275abf92bad4b`).
- PR #147 transaction-time POS catalog authority (`ae74b3fc4738a00b4b64a4e1ac9a545bdbdcf99c`); exact-head Actions run `33778925260` succeeded.
- PR #148 POS location/table/product boundaries (`bb601140f859c4944f5eaae47c907efcd4d8526f`); exact-head Actions run `33795725978` succeeded.
- PR #150 POS idempotency intent binding (`7bc3f142b7db074843016cd01d21eae070041717`).
- PR #151 dine-in location/table/branch-product boundaries (`f2002fd5681aaa5572c84672ded3adc23c511d72`); exact-head Actions run `33819904137` succeeded.
- PR #152 legacy/locationless dine-in product boundary (`e2f9a8e29cf760a2b43e0f2f3429a87ad296391a`); exact-head Actions run `33821697996` succeeded through tests and database recovery drill.
- PR #154 POS global catalog boundary (`8d1762f707938c648189a19da044ce354c29d32d`); exact-head full CI + recovery verification completed before merge.
- PR #155 BusinessTaxPreset tenant boundary (`bdbcad1947fbc7db9b9a1c5c750794fbfac36583`); exact-head run `33822836242` succeeded through tests and database recovery drill.
- PR #157 branch-aware BusinessProduct listing (`4d7d119f5122ab07e51f645b636c961f7bd60ab7`).
- PR #158 dine-in permission + trusted tenant scope (`55f1166a6e17364797c4b1a36981032c1679aa4c`); exact-head run `33829241262` succeeded through tests and database recovery drill.
- PR #159 BusinessTaxPreset default application to dine-in (`07e20fee3dab743d7e113500ce1d08b169d025b2`); exact-head run `33830884690` succeeded through tests and database recovery drill.
- PR #160 public dine-in menu location boundary (`35ee92840527f1743b4974db72b5fdfd708bf5e8`); exact-head run `33831023259` succeeded through tests and database recovery drill.
- PR #161 paid-replay stale-tab closure recovery (`590c9229226e966c29d45b1968102b052140d4e9`); exact-head run `33831227662` succeeded through tests and database recovery drill.
- PR #162 shared dine-in tab read authorization (`2524f6f649f5785fbd5566669760372962ca5c64`) merged after exact-head Actions run `33838812996` passed both the full test suite and database backup/restore drill. The earlier exact-head run `33836614166` had failed only on a stale test fixture; the fixture was corrected and the replacement run passed before merge.

## 3. Cross-repo dine-in convergence already implemented

### Backend
- `BusinessTaxPreset.isDefault` is now consulted by invoice creation only when `taxLines` is omitted; explicit `taxLines: []` remains an intentional tax-free contract.
- Public `/api/business/:bizId/menu` supports an explicit active `locationId`, keeps global products/sections visible, and excludes entries belonging to other branches.
- Paid replay recovery now closes a stale `FINALIZED` dine-in tab without rerunning financial settlement.
- Shared tab reads are now server-side owner-bound: either the requesting customer owns the tab or the caller has trusted business scope plus canonical `restaurant.dinein.manage` permission.

### Business Portal
- PR #49 (`666e2a6adbaa6508a85936f67a4f556163c88db8`) is merged on `main`; push CI run `33836121232` passed. The branch-aware operations flow sends location/table context when opening tabs, uses location-aware product queries, and no longer submits a client-owned tax rate on finalization.
- Superseded draft PR #48 was closed after #49 landed to keep the review surface unambiguous.
- The converged UI derives branch context from selected-tab details, but post-open query-state behavior still needs deliberate lifecycle/realtime verification so a cleared create-form location cannot transiently broaden the menu query while tab detail is loading.

### Flutter customer app
- Main includes PR #79 (`e91e247e559ac9f0168ebd57a64e8c96a965c434`); exact-head Flutter Quality run `33831102056` passed analysis and tests.
- Dine-in ordering now passes the existing tab `locationId` to the public menu endpoint and tolerates the canonical `uncategorised` menu response while preserving legacy compatibility.

## 4. Immediate P0 queue

1. Trace the dine-in lifecycle across business/customer clients for FINALIZED/CLOSED semantics, tip replay, timeout recovery, authoritative refresh, and realtime event ordering now that shared reads are tenant-bound.
2. Trace every `updateAccruedWages()` producer/consumer/history path before modifying payroll accounting; separately prove the period-vs-cumulative accrual behavior around payroll processing/disbursement before changing money movement.
3. Strengthen Admin financial mutation and optimistic pending-queue coverage beyond the existing facade contract tests, then execute tenant/state/realtime, production-operations, load and red-team waves.
4. Reconcile Planning after every verified merge and do not promote in-flight or failed work into the verified baseline.

## 5. Residual risks

- Business Portal branch-aware convergence is merged, but selected-tab/location state and realtime refresh still need a final race audit.
- Flutter customer dine-in now requests branch-aware menus, but broader entry-point/retry/realtime behavior still needs cross-screen reconciliation.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in production operations/load/red-team evidence remain open.
- Payroll still contains both a canonical clock-out accrual path and a legacy `updateAccruedWages()` implementation; callers and accounting semantics must be proven before removal or consolidation.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. Stale backend PRs #149 and #153 remain closed/superseded. Superseded Business Portal draft PR #48 is closed. Backend PRs #159–#162, Business Portal #45 and #49, and frontend #79 are verified/merged and recorded here. The previous PR #162 failure was a test-fixture issue and was not promoted into the verified baseline; replacement exact-head run `33838812996` is the authoritative verification. Keep this document synchronized with every verified cross-repo change.