# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `f8c433be22fa29dd7e0ce7134257020aa56736a8` — includes OrderTracking atomic initialization/serialization, storefront availability and Studio hardening, BusinessTaxPreset default-authority serialization + database uniqueness migration, provider-attempt reference integrity, and dine-in item/finalization serialization. Backend PR #222 and #225 are now merged after exact-head CI passed.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization; Flutter PR #88 and dine-in payment convergence PR #89 are merged.
- Business Portal main: latest verified main includes #81 API/storefront hardening; no open PR.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR.
- Planning main: active loop/ledger reconciled to this state.

## Verified recent work

Backend PR #222 (`fix(finance): bind provider references to one transaction`) exact-head Actions run #855 passed all test and database recovery steps before merge. Backend PR #225 (`fix(dine-in): serialize item mutations with finalization`) was corrected after its first failed suite and its exact-head run #856 passed before merge at `bb542f68404e095ac0438be0747d73ebeb23cc05`. Flutter PR #89 exact-head Flutter Quality passed before merge at `362594350127acf2a9ae55695d2320c41c729e33`.

## Active unresolved work

No currently verified financial/dine-in PR is blocked. The previous #222 failure (`33943464182`) was reproduced as stale provider-attempt fixture expectations; the corrected fixtures use the canonical `TransactionHistory` id and exact-head run #855 passed. The previous #225 failure (`#852`) was reproduced as two legacy adapter expectations that had not been updated for the new atomic mutation boundary; corrected test wiring passed on exact head.

## Next P0 sequence

1. Complete cross-client dine-in lifecycle proof: finalize → invoice creation → payment → CLOSED, including idempotent replay, tips, duplicate/concurrent payment attempts, timeout/reconnect/background recovery, and Business Portal/Admin visibility.
2. Complete POS/invoice tax-authority producer/consumer audit now that `BusinessTaxPreset` default authority is transactional and database-unique.
3. Harden canonical business-invoice creation idempotency/replay, after mapping all runtime callers and preserving tenant/customer binding.
4. Continue financial/control-plane integrity across withdrawals, wallet, escrow, trades, payroll/EWA, refunds/voids, reservations and admin approvals.
5. Complete tenant/actor/state-machine coverage across remaining Business OS resources.
6. Expand Admin financial mutation and operational coverage.
7. Production readiness: deployment/configuration separation, secrets rotation, migration rollback, observability, worker recovery, anomaly/reconciliation alerts.
8. Concurrency/load/adversarial waves and release rehearsal.

## Important discovered contract

`DineInService.finalizeTab()` computes subtotal/grand total from authoritative tab items and transitions OPEN→FINALIZED transactionally; `confirmAndPay()` creates/sends the invoice as needed, pays through the canonical invoice service and transaction-scoped Prisma client, then CAS-closes FINALIZED→CLOSED. Flutter's customer tab is intentionally API-authoritative with socket signals plus 10-second polling, and PR #89 adds an authoritative reread after ambiguous payment errors.

The remaining dine-in audit must verify that Business Portal and Admin Portal observe the same authoritative invoice/payment/tip state without introducing second sources of truth.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.
