# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `ae74b3fc4738a00b4b64a4e1ac9a545bdbdcf99c`; PRs #144–#147 merged | POS location/table/product boundaries; idempotency intent; tax authority; dine-in convergence |
| `AZM-adminPortal` | Main lineage includes prior PR #95 work | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44 dine-in customer-payment authority fix | Cross-repo `confirmAndPay` reconciliation |
| `AZM-frontend` | Main includes verified PR #78 payment-failure truthfulness | Dine-in retry/realtime convergence |
| `AZM-Planning` | Reconciliation branch records this continuation state | Merge this navigation update, then continue active P0 loop |

## 2. Verified backend hardening now on main

- PR #131 payroll Serializable retry.
- PR #132 shift lifecycle-status mutation boundary.
- PR #135 dine-in settlement/finalization/payment-idempotency hardening.
- PR #136 KYB fail-closed gate.
- PR #137 canonical Business OS Finance runtime/routing repair.
- PR #141 inventory restock atomicity (`a0876b2f61d5bc73acb1a1d76368e019d079fe82`).
- PR #143 dine-in test mock correction.
- PR #144 kiosk scoped capability + tenant/employee/user/shift/location binding + PIN rate limit (`22ee6db6633322bca8be3c60a346f974e5e9323c`).
- PR #145 POS server-authoritative settlement and inventory atomicity (`301a5795898f4b7de3b69c8156afe027f82e5155`).
- PR #146 duplicate-line recipe consumption (`2465c01b1746cec95cdec5adcd0275abf92bad4b`).
- PR #147 transaction-time POS catalog authority (`ae74b3fc4738a00b4b64a4e1ac9a545bdbdcf99c`); exact-head Actions run `33778925260` succeeded through tests and database recovery drill.

## 3. Active implementation — PR #148

**POS location/table/product boundary hardening**

- Branch: `fix/pos-location-table-product-boundaries-v3`.
- Head: `449464c4d236b13f8a7210c90f479431e0909f46`.
- Exact-head Actions run: `33779408172`, currently running.
- Settlement validates an active business-owned location; tableId requires locationId and the table must belong to that location; branch-scoped products are restricted to the requested branch; globally available products remain usable.
- Explicit cash customerId is normalized and must reference an existing user.
- Do not merge until exact-head tests and recovery drill are green.

## 4. Immediate P0 queue after #148

1. Bind POS idempotency replay to a canonical request fingerprint/intent.
2. Trace and centralize the actual POS tax authority using `BusinessInvoice`, `BusinessInvoiceTaxLine`, `BusinessTaxPreset` and their producers/consumers; do not replace the legacy 2.5% until proved canonical.
3. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, including tab closure, payment authority, tips, idempotency, realtime and timeout recovery.
4. Trace `updateAccruedWages()` producers/consumers before changing/removing it.
5. Strengthen Admin financial mutation and optimistic pending-queue coverage.
6. Execute tenant/state/realtime matrices, production operations, load and red-team waves.

## 5. Residual risks

- Idempotency keys are unique and tenant-checked but are not yet bound to request payload intent.
- POS still uses a legacy 2.5% tax calculation pending authoritative tax/invoice tracing.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in realtime/reconciliation and production operations/load/red-team evidence remain open.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. PR #34 carried the previous reconciliation. This branch updates the navigation ledger again after PR #147 and while PR #148 is in-flight.
