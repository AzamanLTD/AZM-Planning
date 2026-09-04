# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `f2002fd5681aaa5572c84672ded3adc23c511d72`; PRs #144–#151 merged | PR #152 legacy locationless-tab boundary; POS tax authority |
| `AZM-adminPortal` | Main lineage includes prior PR #95 work | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44 dine-in customer-payment authority fix | Cross-repo invoice/tax and `confirmAndPay` reconciliation |
| `AZM-frontend` | Main includes verified PR #78 payment-failure truthfulness | Dine-in retry/realtime convergence |
| `AZM-Planning` | This reconciliation branch records backend PRs #150/#151 and active #152 | Must merge reconciliation after verified backend work |

## 2. Verified backend hardening now on main

- PR #131 payroll Serializable retry.
- PR #132 shift lifecycle-status mutation boundary.
- PR #135 dine-in settlement/finalization/payment-idempotency hardening.
- PR #136 KYB fail-closed gate.
- PR #137 canonical Business OS Finance routing/runtime repair.
- PR #141 inventory restock atomicity (`a0876b2f61d5bc73acb1a1d76368e019d079fe82`).
- PR #143 dine-in test mock correction.
- PR #144 kiosk scoped capability + tenant/employee/user/shift/location binding + PIN rate limit (`22ee6db6633322bca8be3c60a346f974e5e9323c`).
- PR #145 POS server-authoritative settlement and inventory atomicity (`301a5795898f4b7de3b69c8156afe027f82e5155`).
- PR #146 duplicate-line recipe consumption (`2465c01b1746cec95cdec5adcd0275abf92bad4b`).
- PR #147 transaction-time POS catalog authority (`ae74b3fc4738a00b4b64a4e1ac9a545bdbdcf99c`); exact-head Actions run `33778925260` succeeded through tests and database recovery drill.
- PR #148 POS location/table/product boundaries (`bb601140f859c4944f5eaae47c907efcd4d8526f`); exact-head Actions run `33795725978` succeeded.
- PR #150 POS idempotency intent binding merged as `7bc3f142b7db074843016cd01d21eae070041717`; exact-head gate was completed before merge.
- PR #151 dine-in location/table/branch-product boundary hardening merged as `f2002fd5681aaa5572c84672ded3adc23c511d72`; exact-head Actions run `33819904137` succeeded through full tests and database recovery drill.

## 3. Active implementation — PR #152

**Dine-in legacy locationless-product boundary**

- Branch: `fix/dine-in-legacy-product-boundary`.
- Head: `6de089f1b319f573aaec505b588bc08fb7076e37`.
- Exact-head Actions run: `33820831050`, currently in progress.
- Location-bound tabs resolve only global or exact-location products; legacy tabs with `locationId = null` resolve only global products. This closes the compatibility-path escape where an old tab could resolve a branch-specific product without branch context.

## 4. Immediate P0 queue after #152

1. Trace and centralize the actual POS tax authority using `BusinessInvoice`, `BusinessInvoiceTaxLine`, `BusinessTaxPreset` and their producers/consumers; do not replace the legacy 2.5% until proved canonical.
2. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, including authority, idempotency, tab closure, tips, realtime and timeout recovery.
3. Trace `updateAccruedWages()` producers/consumers before changing/removing it.
4. Strengthen Admin financial mutation and optimistic pending-queue coverage.
5. Execute tenant/state/realtime matrices, production operations, load and red-team waves.

## 5. Residual risks

- PR #152 must not merge until exact-head full tests + database recovery drill are green.
- POS still uses a legacy 2.5% tax calculation pending authoritative tax/invoice tracing.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in cross-repo realtime/reconciliation and production operations/load/red-team evidence remain open.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. PR #149 remains closed/superseded by clean post-#148 PR #150. Backend main has advanced through PR #151; this branch must be merged to keep the canonical continuation state synchronized. PR #152 remains gated on exact-head CI.
