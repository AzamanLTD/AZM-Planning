# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main includes verified PRs #144–#163 plus merged PR #165 realtime hardening | Backend PR #166 stale-payroll snapshot guard is under exact-head CI |
| `AZM-adminPortal` | Main includes PR #95 and merged financial hardening PR #96 | Broaden financial tenant/state/realtime and operational evidence |
| `AZM-businessPortal` | Main includes merged PR #44, #45, #49 and realtime/location race fix #50 | Production event ordering, multi-tab behavior and deeper lifecycle verification |
| `AZM-frontend` | Main includes verified PRs #78, #79, #80 and merged socket-convergence PR #81 | Cross-client lifecycle verification and reconnect/background edge cases |
| `AZM-Planning` | Navigation ledger synchronized through Backend #165, Business Portal #50, Frontend #81 and Admin #96 | Reconcile after every verified backend/payment change |

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
- PR #152 legacy/locationless dine-in product boundary (`e2f9a8e29cf760a2b43e0f2f3429a87ad296391a9`); exact-head run `33821697996` succeeded through tests and database recovery drill.
- PR #154 POS global catalog boundary (`8d1762f707938c648189a19da044ce354c29d32d`); exact-head full CI + recovery verification completed before merge.
- PR #155 BusinessTaxPreset tenant boundary (`bdbcad1947fbc7db9b9a1c5c750794fbfac36583`); exact-head run `33822836242` succeeded through tests and database recovery drill.
- PR #157 branch-aware BusinessProduct listing (`4d7d119f5122ab07e51f645b636c961f7bd60ab7`).
- PR #158 dine-in permission + trusted tenant scope (`55f1166a6e17364797c4b1a36981032c1679aa4c`); exact-head run `33829241262` succeeded through tests and database recovery drill.
- PR #159 BusinessTaxPreset default application to dine-in (`07e20fee3dab743d7e113500ce1d08b169d025b2`); exact-head run `33830884690` succeeded through tests and database recovery drill.
- PR #160 public dine-in menu location boundary (`35ee92840527f1743b4974db72b5fdfd708bf5e8`); exact-head run `33831023259` succeeded through tests and database recovery drill.
- PR #161 paid-replay stale-tab closure recovery (`590c9229226e966c29d45b1968102b052140d4e9`); exact-head run `33831227662` succeeded through tests and database recovery drill.
- PR #162 shared dine-in tab read authorization (`2524f6f649f5785fbd5566669760372962ca5c64`) merged after exact-head Actions run `33838812996` passed both the full test suite and database backup/restore drill.
- PR #163 business dine-in queue lifecycle filtering (`b0e23c692b2cc372d84daa557282f4644f36fd52`) merged after exact-head Actions run `33839219871` passed the full test suite and database backup/restore drill. With no explicit status, business tab reads now return only `OPEN` and `FINALIZED`; explicit status filters remain authoritative, including `CLOSED`.
- PR #165 dine-in business realtime lifecycle events (`c3af291a5eff47d8e31ae703953346d201d09f7a`) merged after exact-head Actions run `33841251654` passed dependencies, schema application, tests, and the database backup/restore drill. The failed predecessor was closed after a branch-reset schema corruption was repaired; the merged retry restores the full Prisma schema and uses only schema-supported notification types.

## 3. Cross-repo dine-in convergence already implemented

### Backend
- `BusinessTaxPreset.isDefault` is now consulted by invoice creation only when `taxLines` is omitted; explicit `taxLines: []` remains an intentional tax-free contract.
- Public `/api/business/:bizId/menu` supports an explicit active `locationId`, keeps global products/sections visible, and excludes entries belonging to other branches.
- Paid replay recovery now closes a stale `FINALIZED` dine-in tab without rerunning financial settlement.
- Shared tab reads are now server-side owner-bound: either the requesting customer owns the tab or the caller has trusted business scope plus canonical `restaurant.dinein.manage` permission.
- Business tab queue reads now have lifecycle-safe defaults: no-status queries expose only actionable `OPEN` and `FINALIZED` tabs, while explicit status requests remain deterministic.
- Business queue lifecycle events now persist through the canonical notification service and emit the existing `biz_notification` socket contract for tab opened/finalized/paid events; notification failure remains non-fatal to the underlying transaction.
- The canonical dine-in service emits customer-specific `dine_in_tab_opened`, `dine_in_item_added`, `dine_in_tab_finalized`, and `dine_in_tab_paid` Socket.IO events to the customer user room.

### Business Portal
- PR #49 (`666e2a6adbaa6508a85936f67a4f556163c88db8`) is merged on `main`; push CI run `33836121232` passed.
- PR #50 merged as `05f3b3ef84e0308fd51fdfb531e4a7d97865b5c7`; exact-head run `33844587449` completed dependencies, smoke test, tests and build successfully. It closes the selected-tab/location race and invalidates DineInV2 on canonical lifecycle events.

### Flutter customer app
- PR #79 (`e91e247e559ac9f0168ebd57a64e8c96a965c434`) passed exact-head Flutter Quality run `33831102056`.
- PR #80 merged as `739c03df714967f27c4b2f17d1741793f6ca737d` after exact-head run `33844855657` passed analysis, tests with coverage and coverage upload. The tab screen now performs authoritative active/resume refresh.
- PR #81 (`fix/dinein-socket-convergence`) merged as `45bd202ae9fb5581f350e9518b6879e150de17ff` after exact-head Flutter Quality run `33846824812` passed analysis, tests and coverage. The customer provider now listens through the existing unified SocketService, reloads only on matching tab events, and retains polling/resume resilience.

### Admin Portal
- PR #96 (`fix(financial): make withdrawal optimistic queue concurrency-safe`) merged as `44b3588b5a134b84a528c533e39ba36b9a9c297f` after CI run `33852179965` completed lint, typecheck and build successfully. The withdrawal queue now performs targeted optimistic updates/rollbacks, preserves query metadata, and avoids overwriting newer realtime/refetched state during rollback; batch approval applies the same per-item protection.

## 4. Immediate P0 queue

1. Finish Backend PR #166 stale payroll protection: exact-head CI must pass the full test suite and database recovery drill before merge. Then prove `updateAccruedWages()` producers/consumers/history and period-vs-cumulative semantics before any further payroll consolidation.
2. Complete the cross-client dine-in lifecycle audit using the now-merged business/customer realtime paths: FINALIZED/CLOSED semantics, tip replay, reconnect/background ordering, multi-tab behavior, timeout recovery and authoritative refresh.
3. Expand Admin financial mutation coverage into tenant/state/realtime and production-operations/load/red-team waves beyond the optimistic withdrawal queue work already merged in #96.
4. Reconcile Planning after every verified merge and never promote in-flight or failed work into the verified baseline.

## 5. Residual risks

- Dine-in production event ordering and multi-tab/end-to-end behavior still need stronger evidence despite business and customer realtime convergence being implemented.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in production operations/load/red-team evidence remain open.
- Payroll still contains both the canonical clock-out accrual mutation and legacy `updateAccruedWages()`; PR #166 protects disbursement against stale prepared records but broader semantic consolidation requires proof.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. Stale backend PRs #149 and #153 remain closed/superseded. Superseded Business Portal draft PR #48 is closed. Backend PRs #159–#163 and #165, Business Portal #45, #49 and #50, frontend #79, #80 and #81, and Admin #96 are verified/merged and recorded here. Backend PR #166 is intentionally in flight until exact-head CI and database recovery evidence complete. The previous PR #162 failure was a test-fixture issue and was not promoted into the verified baseline; replacement exact-head run `33838812996` is authoritative. PR #163 exact-head run `33839219871` and PR #165 exact-head run `33841251654` are authoritative verification evidence for those changes. The prior PR #164 attempt was closed after its branch reset exposed and repaired a mistaken partial Prisma schema replacement; no broken schema change is retained. Backend duplicate PR #167 was closed without merge because #166 is the canonical stale-payroll implementation. Keep this document synchronized with every verified cross-repo change.
