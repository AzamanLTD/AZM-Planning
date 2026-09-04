# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main includes verified PRs #144–#163 plus merged PR #165 realtime hardening | Cross-client dine-in lifecycle/realtime verification |
| `AZM-adminPortal` | Main lineage includes PR #95 and prior financial hardening | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44, #45 and branch-aware dine-in convergence PR #49 plus realtime/location race fix #50 | Verify post-convergence lifecycle/realtime behavior against the customer app and backend |
| `AZM-frontend` | Main includes verified PRs #78, #79 and #80; PR #81 is under exact-head CI | Customer-side dine-in realtime convergence |
| `AZM-Planning` | Navigation ledger synchronized through Backend #165, Business Portal #50 and Frontend #80; PR #81 remains in flight | Keep navigation synchronized after each verified merge |

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
- PR #162 shared dine-in tab read authorization (`2524f6f649f5785fbd5566669760372962ca5c64`) merged after exact-head Actions run `33838812996` passed both the full test suite and database backup/restore drill. The earlier exact-head run `33836614166` had failed only on a stale test fixture; the fixture was corrected and the replacement run passed before merge.
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
- The canonical dine-in service also emits customer-specific `dine_in_tab_opened`, `dine_in_item_added`, `dine_in_tab_finalized`, and `dine_in_tab_paid` Socket.IO events to the customer user room. These are useful convergence signals; authoritative tab/payment state remains API-backed.

### Business Portal
- PR #49 (`666e2a6adbaa6508a85936f67a4f556163c88db8`) is merged on `main`; push CI run `33836121232` passed. The branch-aware operations flow sends location/table context when opening tabs, uses location-aware product queries, and no longer submits a client-owned tax rate on finalization.
- PR #50 is now merged as `05f3b3ef84e0308fd51fdfb531e4a7d97865b5c7`; exact-head Business Portal CI run `33844587449` completed every validate step successfully (dependencies, smoke test, tests, and build). It separates selected-tab branch context from create-form state and expands dine-in realtime invalidation to the exact DineInV2 cache roots (`openTabs`, `dineInTab`) so canonical business lifecycle notifications immediately refetch the authoritative projections.
- Superseded draft PR #48 was closed after #49 landed to keep the review surface unambiguous.

### Flutter customer app
- Main includes PR #79 (`e91e247e559ac9f0168ebd57a64e8c96a965c434`); exact-head Flutter Quality run `33831102056` passed analysis and tests.
- Dine-in ordering passes the existing tab `locationId` to the public menu endpoint and tolerates the canonical `uncategorised` menu response while preserving legacy compatibility.
- PR #80 merged as `739c03df714967f27c4b2f17d1741793f6ca737d` after exact-head Flutter Quality run `33844855657` completed analysis, tests with coverage, and coverage upload successfully. The customer tab screen now invalidates the canonical tab query every 10 seconds while mounted and immediately on app resume, with timer/observer cleanup, to converge after waiter-side finalization and background/reconnect gaps.
- PR #81 (`fix/dinein-socket-convergence`) remains in flight under exact-head Flutter Quality run `33846674689`. It uses the existing unified SocketService singleton to consume customer-specific dine-in lifecycle events and trigger an authoritative tab API reload only when the event `tabId` matches the provider's active tab; polling/resume refresh remains as resilience.

## 4. Immediate P0 queue

1. Complete the remaining cross-client dine-in lifecycle audit: verify customer-specific socket events against API refresh, FINALIZED/CLOSED semantics, tip replay, timeout/reconnect behavior, and multi-tab behavior; merge #81 only after exact-head CI is green and reconcile Planning immediately.
2. Finish the payroll accounting proof and stale-snapshot hardening on Backend PR #166: verify all `updateAccruedWages()` producer/consumer/history paths, period-vs-cumulative semantics, and merge only after full tests plus database recovery evidence.
3. Strengthen Admin financial mutation and optimistic pending-queue coverage beyond the existing facade contract tests, then execute tenant/state/realtime, production-operations, load and red-team waves.
4. Reconcile Planning after every verified merge and do not promote in-flight or failed work into the verified baseline.

## 5. Residual risks

- Business Portal branch-aware convergence and realtime cache invalidation are now hardened, but production event ordering and multi-tab UI behavior still require end-to-end verification.
- Flutter customer dine-in has authoritative active/resume refresh from PR #80 and direct SocketService convergence is in PR #81 pending exact-head verification.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in production operations/load/red-team evidence remain open.
- Payroll still contains both a canonical clock-out accrual path and a legacy `updateAccruedWages()` implementation; PR #166 adds a disbursement snapshot guard but caller/semantic proof is still required before broader consolidation.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. Stale backend PRs #149 and #153 remain closed/superseded. Superseded Business Portal draft PR #48 is closed. Backend PRs #159–#163 and #165, Business Portal #45, #49 and #50, and frontend #79 and #80 are verified/merged and recorded here. Frontend PR #81 is intentionally recorded as in flight until exact-head CI completes. The previous PR #162 failure was a test-fixture issue and was not promoted into the verified baseline; replacement exact-head run `33838812996` is the authoritative verification. PR #163 exact-head run `33839219871` and PR #165 exact-head run `33841251654` are the authoritative verification evidence for those changes. The prior PR #164 attempt was closed after its branch reset exposed and repaired a mistaken partial Prisma schema replacement; no broken schema change is retained. Keep this document synchronized with every verified cross-repo change.
