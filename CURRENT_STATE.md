# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `bb601140f859c4944f5eaae47c907efcd4d8526f`; PRs #144–#148 merged | POS idempotency intent; tax authority; dine-in convergence |
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
- PR #148 POS location/table/product boundaries (`bb601140f859c4944f5eaae47c907efcd4d8526f`); exact-head Actions run `33795725978` succeeded.

## 3. Active implementation — PR #149

**POS idempotency intent binding**

- Branch: `fix/pos-idempotency-intent-binding`.
- Head: `fe589f0039cf6055aaa03207a44ba7f095d85ec0`.
- Exact-head Actions run: `33795907968`, currently in progress.
- The implementation derives a canonical request fingerprint, persists it in POS ledger metadata, rejects a mismatched reuse, and permits legacy replay when no fingerprint exists.
- PR #149 must be rebased/recreated from the post-#148 main before merge because it was opened against the pre-#148 base and touches the same POS service.

## 4. Immediate P0 queue after #149

1. Trace and centralize the actual POS tax authority using `BusinessInvoice`, `BusinessInvoiceTaxLine`, `BusinessTaxPreset` and their producers/consumers; do not replace the legacy 2.5% until proved canonical.
2. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, including tab closure, payment authority, tips, idempotency, realtime and timeout recovery.
3. Trace `updateAccruedWages()` producers/consumers before changing/removing it.
4. Strengthen Admin financial mutation and optimistic pending-queue coverage.
5. Execute tenant/state/realtime matrices, production operations, load and red-team waves.

## 5. Residual risks

- PR #149 is intentionally not mergeable yet because its base predates merged #148; the logical fix must be recreated/rebased on post-#148 main and then re-gated.
- POS still uses a legacy 2.5% tax calculation pending authoritative tax/invoice tracing.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in realtime/reconciliation and production operations/load/red-team evidence remain open.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. This reconciliation records the verified #148 merge and the active #149 loop.
