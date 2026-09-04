# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-04 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `e2f9a8e29cf760a2b43e0f2f3429a87ad296391a`; PRs #144–#152 merged | Tax preset tenant boundary; POS global/branch boundary |
| `AZM-adminPortal` | Main lineage includes prior PR #95 work | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44 dine-in customer-payment authority fix | Dine-in location-aware UX and invoice/tax convergence |
| `AZM-frontend` | Main includes verified PR #78 payment-failure truthfulness | Dine-in retry/realtime convergence |
| `AZM-Planning` | This reconciliation records verified backend #152 and active backend #154/#155 | Keep navigation synchronized after each verified merge |

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

## 3. Active implementations

### Tax preset tenant boundary — PR #155

- Branch: `fix/tax-preset-tenant-boundary-v2`.
- Head: `ca553255a4ae177f46f8fdd8cace878ce4c542aa`.
- Production defect: BusinessTaxPreset PATCH previously updated by bare preset ID after only tenant-scoping the default-reset query; the new middleware rejects a preset outside the caller's business before the handler runs.
- Exact-head CI is required before merge.
- Superseded stale PR #153 was closed after PR #152 changed main; the implementation was recreated cleanly on current main.

### POS global catalog boundary — PR #154

- Branch: `fix/pos-global-product-boundary`.
- Head: `2a97cdb52f27f1d2bd003cb1fba709364141e71f`.
- Production hardening: locationless POS requests now resolve only globally scoped products; explicit locations still allow global or exact-location products.
- Exact-head CI is required before merge.

## 4. Immediate P0 queue after #154/#155

1. Complete tax authority trace: establish whether BusinessTaxPreset is a template/convenience layer or runtime authority, and remove/replace legacy POS 2.5% only after the producer/consumer trace proves the canonical policy.
2. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, especially location/table context, tax behavior, tab closure after payment replay, tips, idempotency and timeout recovery.
3. Bring Business Portal dine-in UI/API onto the location-aware contract; current UI still opens tabs without location context and exposes legacy tax-rate UI that the backend adapter discards.
4. Trace every `updateAccruedWages()` producer/consumer/history before removal or restriction.
5. Strengthen Admin financial mutation and optimistic pending-queue coverage, then execute tenant/state/realtime, production-operations, load and red-team waves.

## 5. Residual risks

- PR #155 and PR #154 are pending exact-head full CI + database recovery verification.
- BusinessTaxPreset is currently used by Business Portal as a selectable tax template, while invoice persistence is authoritative on BusinessInvoiceTaxLine rows; the canonical relationship to POS taxation is not yet established.
- Dine-in location-aware server boundaries are enforced, but Business Portal still lacks a location/table selector and therefore can create legacy/locationless tabs.
- Kiosk PIN protection is locally rate-limited; distributed topology and enumeration/timing behavior remain open.
- Dine-in realtime/reconciliation and production operations/load/red-team evidence remain open.

## 6. Planning hygiene

Stale Planning PR #27 remains closed/superseded. Stale backend PR #149 and tax PR #153 are closed/superseded. Backend main is now `e2f9a8e29cf760a2b43e0f2f3429a87ad296391a` after verified PR #152. This Planning branch records the verified #152 merge and active #154/#155 work; it should be merged after the repository changes are reviewed.
