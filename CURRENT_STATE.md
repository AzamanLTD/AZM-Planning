# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main includes verified PRs #144–#155, #157–#161 | Cross-repo dine-in UX convergence |
| `AZM-adminPortal` | Main lineage includes PR #95 and prior financial hardening | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44 and #45 dine-in permission-contract fix | Location/table UX and misleading tax-rate controls |
| `AZM-frontend` | Main includes verified PRs #78 and #79 | Dine-in retry/realtime convergence |
| `AZM-Planning` | Reconciled through backend #161 and frontend #79 | Keep navigation synchronized after each verified merge |

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

## 3. Cross-repo dine-in convergence already implemented

### Backend
- `BusinessTaxPreset.isDefault` is now consulted by invoice creation only when `taxLines` is omitted; explicit `taxLines: []` remains an intentional tax-free contract.
- Public `/api/business/:bizId/menu` supports an explicit active `locationId`, keeps global products/sections visible, and excludes entries belonging to other branches.
- Paid replay recovery now closes a stale `FINALIZED` dine-in tab without rerunning financial settlement.

### Flutter customer app
- Main includes PR #79 (`e91e247e559ac9f0168ebd57a64e8c96a965c434`); exact-head Flutter Quality run `33831102056` passed analysis and tests.
- Dine-in ordering now passes the existing tab `locationId` to the public menu endpoint and tolerates the canonical `uncategorised` menu response while preserving legacy compatibility.

## 4. Immediate P0 queue

1. Finish Business Portal dine-in convergence: wire actual location/table selection into `openDineInTab`, make product selection branch-aware, and remove the misleading client-owned tax-rate path now that invoice tax authority lives server-side.
2. Trace dine-in lifecycle across business/customer clients for FINALIZED/CLOSED semantics, tip replay, timeout recovery, and realtime reconciliation after the paid-replay fix.
3. Trace every `updateAccruedWages()` producer/consumer/history path before modifying payroll accounting.
4. Strengthen Admin financial mutation and optimistic pending-queue coverage, then execute tenant/state/realtime, production-operations, load and red-team waves.

## 5. Residual risks

- Business Portal still does not provide complete location/table selection in the current UI; its dine-in product list is not branch-scoped.
- Business Portal still exposes `taxRatePct` as a client control during tab finalization even though dine-in invoice tax is now server-authoritative via `BusinessTaxPreset`.
- Flutter customer dine-in now requests branch-aware menus, but broader entry-point/retry/realtime behavior still needs cross-screen reconciliation.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in production operations/load/red-team evidence remain open.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. Stale backend PRs #149 and #153 remain closed/superseded. Backend PRs #159, #160 and #161 and Business Portal #45/frontend #79 are now verified/merged and recorded here. Keep this document synchronized with every verified cross-repo change.