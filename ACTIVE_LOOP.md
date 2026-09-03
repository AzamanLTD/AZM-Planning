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
- Backend PR #144 kiosk scoped capability + PIN rate limiting merged at current-main base.
- Backend PR #145 POS settlement + transaction-time inventory consumption merged at current-main base.
- Frontend PR #78 dine-in payment failure truthfulness merged/verified.
- Stale Planning PR #27 was closed after becoming obsolete.

### Backend main

`301a5795898f4b7de3b69c8156afe027f82e5155`

### Active implementation

#### POS duplicate-line recipe consumption — PR #146

- Branch: `fix/pos-recipe-duplicate-line-consumption`
- Head: `ba4fc6912b35f456cfdde0161a0fb08a36ace230`
- PR: #146
- Fix: repeated lines for the same menu product previously used a last-value-wins `Map`, under-consuming recipe ingredients. The implementation now sums quantities by product before recipe requirements are calculated.
- Regression test added for two duplicate lines of the same restaurant product.
- Exact-head Actions gate is required before merge.

### Next P0 after #146

1. Rework POS authority at the transaction boundary: revalidate catalog/product state inside the Serializable transaction so price/availability cannot become stale between pre-read and settlement.
2. Complete POS location/table ownership and customer-identity integrity review.
3. Define and safely encode idempotency payload semantics; same key with different intent must not silently replay unless the API explicitly defines that behavior.
4. Resolve POS tax authority by tracing actual invoice/tax models and producers; do not assume the legacy 2.5% value is canonical.
5. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility.
6. Trace all `updateAccruedWages()` consumers/history before removal or restriction.
7. Strengthen Admin financial mutation and optimistic pending-queue coverage.
8. Then execute tenant/state/realtime matrices, production operations, load and red-team waves.

## CI / release gate

Backend Actions now executes normally on the public repositories. The required gate remains: exact PR head, full test stage, and database backup/restore drill must be green before merge.

For PR #145, exact-head run `33777699164` completed successfully on head `493d82b32fd8d8d597b6bbccac154a94b2f168e8`, including tests and recovery drill, and PR #145 was then merged.

## Duplicate-work prohibition

Do not resurrect superseded POS #138/#140 or kiosk #142 work, and do not reopen Planning #27. Build from current `main` and consolidate into the canonical active path.

## Continuation rule

When CI is running, continue independent audits instead of waiting idle. When CI completes, return to the exact-head gate. After every verified merge, reconcile `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`, then continue to the next P0.
