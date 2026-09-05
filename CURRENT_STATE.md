# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `7a24a380bf6a5da4641dc4de3bc904ed31476197` — includes OrderTracking atomic initialization/serialization, storefront availability and Studio hardening, provider-attempt identity foundation, and BusinessTaxPreset default-authority serialization + database uniqueness migration. Backend PR #223 CI modernization and #224 tax authority are merged.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization. Flutter PR #88 is merged.
- Business Portal main: latest verified main includes #81 API/storefront hardening; no open PR.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR.
- Planning main: active loop/ledger reconciled to this state.

## Verified recent work

Backend PRs #220, #221, #223 and #224 are verified/merged. #224 exact-head test run #849 passed. Flutter #88 exact-head Flutter Quality run #343 passed before merge.

## Active unresolved work

Backend PR #222 (`fix(finance): bind provider references to one transaction`) remains open and unmerged. Exact-head Actions run `33943464182` reached the database/test phase successfully but failed during `Run tests`; the connector exposes the failed job summary but not its underlying test output. The migration establishes the provider-attempt table/unique provider-reference identity, so the intended correlation invariant remains valid, but the PR must not be promoted until the concrete test failure is deterministically obtained or reproduced.

Flutter PR #89 (`fix/dine-in-payment-replay-convergence`) is open at exact head `e17951afed6c2262436bb90eac239fba2f9aa140`; its Flutter Quality run is still in progress. It changes only the payment error/recovery path so a committed CLOSED tab converges to success after a lost payment response.

## Next P0 sequence

1. Diagnose/fix backend #222 without weakening its invariant.
2. Prove dine-in cross-client lifecycle end-to-end: finalize → invoice creation → payment → CLOSED, including idempotent replay, tips, duplicate/concurrent payment attempts, timeout/reconnect/background recovery, and Business Portal/Admin visibility.
3. Complete POS/invoice tax-authority producer/consumer audit now that BusinessTaxPreset default authority is locked down.
4. Continue financial/control-plane integrity across withdrawals, wallet, escrow, trades, payroll/EWA, refunds/voids, reservations and admin approvals.
5. Complete tenant/actor/state-machine coverage across remaining Business OS resources.
6. Expand Admin financial mutation and operational coverage.
7. Production readiness: deployment/configuration separation, secrets rotation, migration rollback, observability, worker recovery, anomaly/reconciliation alerts.
8. Concurrency/load/adversarial waves and release rehearsal.

## Important discovered contract

`DineInService.finalizeTab()` computes subtotal/grand total from authoritative tab items and transitions OPEN→FINALIZED transactionally; `confirmAndPay()` creates/sends the invoice as needed, pays through the canonical invoice service and transaction-scoped Prisma client, then CAS-closes FINALIZED→CLOSED. Flutter's customer tab is intentionally API-authoritative with socket signals plus 10-second polling. Flutter PR #89 adds an authoritative reread after payment errors to avoid false failure UX when a payment may already be committed.

The remaining dine-in audit must verify that the Business Portal and Admin Portal observe the same authoritative invoice/payment/tip state without introducing second sources of truth.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.
