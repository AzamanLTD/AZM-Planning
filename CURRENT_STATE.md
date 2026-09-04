# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main includes verified PRs #144–#155, #157–#158; PR #159 active | Dine-in tax authority; cross-repo convergence |
| `AZM-adminPortal` | Main lineage includes PR #95 and prior financial hardening | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44 and PR #45 dine-in permission-contract fix | Location/table UX and tax/default convergence |
| `AZM-frontend` | Main includes verified PR #78 payment-failure truthfulness | Dine-in retry/realtime convergence |
| `AZM-Planning` | This reconciliation updates navigation through verified backend #158 and Business Portal #45 | Keep navigation synchronized after each verified merge |

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

## 3. Active implementation

### Dine-in/default invoice tax authority — PR #159

- Branch: `fix/dine-in-default-tax-preset-v3`.
- Head: `9f47d2ff36219504dca6c0d950a0d5a857174c74`.
- Runtime change: `BusinessTaxPreset.isDefault` is now consulted by `createInvoice()` only when tax lines are omitted; explicit `taxLines: []` remains tax-free by contract.
- Dine-in invoice creation no longer sends an explicit empty tax array, so the business default can actually apply.
- Focused regression coverage added for both default-tax and explicit-tax-free behavior.
- Exact-head full CI + database recovery drill required before merge.

## 4. Immediate P0 queue after #159

1. Finish cross-repo dine-in convergence: wire Business Portal location/table selection to the branch-aware API, eliminate the misleading tax-rate UI path, and reconcile Flutter customer flows.
2. Deep-audit `confirmAndPay` replay/closure semantics, including already-paid replay behavior, customer-visible state, tip handling, idempotency and timeout recovery.
3. Trace every `updateAccruedWages()` producer/consumer/history before removal or restriction.
4. Strengthen Admin financial mutation and optimistic pending-queue coverage, then execute tenant/state/realtime, production-operations, load and red-team waves.

## 5. Residual risks

- PR #159 is pending exact-head full CI + database recovery verification.
- BusinessTaxPreset is now a runtime default source for invoices when tax lines are omitted; explicit tax lines still remain caller-authoritative. POS legacy 2.5% behavior has not been replaced without a separate policy trace.
- Dine-in server boundaries and permission gates are enforced, but Business Portal still does not provide complete location/table selection in the current UI.
- Flutter customer dine-in still needs an endpoint/model/context reconciliation against the current branch-aware server contract.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in realtime/reconciliation and production operations/load/red-team evidence remain open.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. Stale backend PRs #149 and #153 remain closed/superseded. Backend PRs #154, #155, #157 and #158 are now verified on main; Business Portal PR #45 is also merged. This reconciliation tracks those merges and the active #159 implementation. Merge this Planning reconciliation after the repository changes are reviewed.
