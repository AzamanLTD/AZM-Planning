# AZAMAN Active Engineering Loop

**Purpose:** persistent continuation state for long-running engineering work. A new session MUST resume from this file when non-empty/active rather than restarting or asking for the task to be restated.

## Operating contract

1. Read `START_HERE.md`, `ROADMAP.md`, `CURRENT_STATE.md`, then this file, then `EXECUTION_LEDGER.json`.
2. Reconcile every referenced repo/PR/branch/SHA against GitHub before changing code.
3. Before creating a branch or PR, search existing branches, open PRs, recent merges and equivalent implementations/tests.
4. Execute the entire active work package: research → trace producers/consumers → inspect schema/authorization/state machine → implement → test → exact-head CI → diff audit → merge → verify `main` → update Planning.
5. Never stop merely because a defect was found, a patch was written, a local test passed, or a PR was opened.
6. If CI fails, diagnose and fix the actual defect; do not weaken tests/type checking.
7. After verified completion, immediately advance to the next unchecked P0 unless genuinely blocked.
8. If blocked, record the exact blocker and smallest required user action. Otherwise continue autonomously.
9. Never recreate completed work or competing implementations.

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Verified on current GitHub main

- Backend PR #131 payroll Serializable retry merged/verified.
- Backend PR #132 shift generic-status mutation boundary merged/verified.
- Backend PR #135 dine-in settlement/finalization/payment-idempotency hardening merged.
- Backend PR #136 KYB gate fail-closed hardening merged.
- Backend PR #137 canonical Business OS Finance routing/runtime repair merged/verified.
- Backend PR #141 inventory restock atomicity merged/verified.
- Backend PR #143 corrected the real dine-in test mock failure.
- Backend PR #144 kiosk scoped capability + PIN rate limiting merged.
- Backend PR #145 POS settlement/inventory atomicity merged.
- Backend PR #146 duplicate-line recipe consumption merged.
- Backend PR #147 transaction-time POS catalog authority merged; exact-head Actions run `33778925260` succeeded through tests and database recovery drill.
- Backend PR #148 POS location/table/product boundary hardening merged as `bb601140f859c4944f5eaae47c907efcd4d8526f`; exact-head Actions run `33795725978` succeeded.
- Frontend PR #78 dine-in payment failure truthfulness merged/verified.
- Business Portal PR #44 customer payment authority fix merged.
- Stale Planning PR #27 closed after becoming obsolete.

### Backend main

`bb601140f859c4944f5eaae47c907efcd4d8526f`

### Active implementation

#### POS idempotency intent binding — PR #149

- Branch: `fix/pos-idempotency-intent-binding`
- Head: `fe589f0039cf6055aaa03207a44ba7f095d85ec0`
- PR: #149
- Fix: derive a canonical request fingerprint from tenant, actor, normalized items, payment inputs, source and location/table/customer context; persist it in the POS ledger metadata; reject a same-key request with a different fingerprint while allowing legacy rows without fingerprints to replay safely.
- Exact-head Actions run `33795907968` is currently in progress; do not merge until the full tests + recovery drill gate is green.

### Next P0 after #149

1. Resolve POS tax authority by tracing `BusinessInvoice`, `BusinessInvoiceTaxLine`, `BusinessTaxPreset` and their actual producers/consumers; do not assume the legacy 2.5% POS tax is canonical.
2. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, including tab closure, payment authority, tips, idempotency, realtime and timeout recovery.
3. Trace every `updateAccruedWages()` producer/consumer/history before removal or restriction.
4. Strengthen Admin financial mutation and optimistic pending-queue coverage.
5. Execute tenant/state/realtime matrices, production operations, load and red-team waves.

## CI / release gate

Backend Actions executes normally on the public repositories. The required gate remains: exact PR head, full test stage, and database backup/restore drill must be green before merge.

## Duplicate-work prohibition

Do not resurrect superseded POS #138/#140 or kiosk #142 work, and do not reopen Planning #27. Build from current `main` and consolidate into the canonical active path.

## Continuation rule

When CI is running, continue independent audits instead of waiting idle. When CI completes, return to the exact-head gate. After every verified merge, reconcile `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`, then continue to the next P0.
