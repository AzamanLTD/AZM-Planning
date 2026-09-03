# AZAMAN Active Engineering Loop

**Purpose:** persistent continuation state for long-running engineering work. A new session MUST resume from this file when it is non-empty/active rather than restarting or asking the user to restate the task.

## Operating contract

1. Read `START_HERE.md`, `ROADMAP.md`, `CURRENT_STATE.md`, then this file, then `EXECUTION_LEDGER.json`.
2. Reconcile every referenced repo/PR/branch/SHA against GitHub before changing code.
3. Before creating a branch or PR, search existing branches, open PRs, recent merges and equivalent implementations/tests.
4. Execute the entire active work package: research → trace producers/consumers → inspect schema/authorization/state machine → implement → test → exact-head CI → diff audit → merge → verify `main` → update this file, `CURRENT_STATE.md` and `EXECUTION_LEDGER.json`.
5. Do not stop merely because a defect was found, a patch was written, a local test passed, or a PR was opened.
6. If CI fails, diagnose and fix it; do not weaken tests/type checking to obtain green CI.
7. After a verified completion, immediately select the next unchecked P0 from `ROADMAP.md` unless genuinely blocked.
8. If blocked by missing access, an external dependency, or a decision that cannot safely be inferred, record the exact blocker and the smallest required user action. Otherwise continue autonomously.
9. Never duplicate completed work. If an existing implementation is partial, continue/consolidate it instead of creating a competing implementation.

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Verified completed in this continuation

- Backend PR #131 payroll Serializable retry verified green and merged.
- Backend PR #132 shift generic-status mutation boundary verified green and merged.
- Backend PR #135 dine-in finalization/item-price/payment-idempotency hardening merged.
- Backend PR #136 KYB gate fail-closed hardening merged.
- Backend PR #137 canonical Business OS Finance runtime/route repair merged.
- Backend PR #141 inventory restock atomicity verified green and merged to current Backend main as `a0876b2f61d5bc73acb1a1d76368e019d079fe82`.
- Backend PR #143 corrected the missing `dineInTabItem` test mock that caused the earlier real Jest failure.
- Frontend PR #78 dine-in payment failure propagation verified and merged.
- Planning persistent continuation files are canonical on Planning main lineage.

### Current active work package: Business OS financial/operational mutation correctness

#### Kiosk — replacement PR #144
- PR #142 is closed/superseded because it was based on the pre-inventory main head.
- PR #144 is open on `fix/business-os-kiosk-capability-v2`, head `4fca0d65a416b85cefa22ec4b15256b5a6cff25d` and targets Backend main `a0876b2f...`.
- Capability signing/verification is isolated; clock-in/out enforce tenant/employee/user/shift/location binding; PIN auth validates business location.
- Additional hardening adds a second-layer `express-rate-limit` on the public PIN challenge: 10 attempts/15 minutes, successful attempts excluded.
- Exact-head run `33776303876` is currently executing the Jest test stage. Do not merge before full green test + recovery drill and final diff review.

#### POS — replacement PR #145
- PR #140 is closed/superseded; PR #138 was already superseded.
- PR #145 is open on `fix/business-os-pos-atomicity-v3`, head `c01ab35692f7cf237387ecf00d5454ad748a2c57`, targeting Backend main `a0876b2f...`.
- Implementation preserves server-authoritative catalog pricing, integer quantities, CASH/AZM/SPLIT settlement, conditional AZM debit, BusinessOrderItem persistence, Serializable retry and tenant-scoped idempotent replay.
- New work closes the prior stock-integrity gap: tracked `BusinessProduct.stockQty` and restaurant `RecipeIngredient` quantities are consumed inside the same transaction as the order and ledger. Transaction-time stock predicates prevent overselling; shared ingredients are aggregated before decrement.
- Remaining semantic risks: legacy-compatible 2.5% tax is not yet proven to be the final tax authority; location/table integrity, customer identity semantics and complete replay payload binding still require evidence.
- Exact-head run is active: `33776598303` on `c01ab356...`. Do not merge until it completes green.

### CI / release-gate evidence

Backend repositories are now public. Current evidence shows GitHub-hosted Actions executing the full canonical workflow normally. Successful exact-head runs `33774674487` (POS predecessor), `33774689580` (kiosk predecessor), and `33774699365` (inventory predecessor) all completed through tests and the database recovery drill.

The earlier inventory failure `33749730364` was a real Jest failure (128 suites passed, 1 failed; 898 tests passed, 1 failed), fixed by PR #143. The old zero-step/startup diagnosis is superseded and must not be carried forward.

## Exact next actions

1. Monitor #144 and #145 until exact-head CI completes; diagnose/fix any failures without weakening the release gate; final-review and merge only when green.
2. Complete POS tax-authority research/trace against `BusinessTaxPreset` and invoice tax semantics, then add the smallest authoritative regression coverage needed.
3. Complete POS location/table and customer identity integrity review.
4. Complete idempotency payload semantics review: decide and document whether replay is allowed for same key with different intent, and encode the policy safely without breaking offline replay.
5. Trace every `updateAccruedWages()` consumer/history before removal/restriction; `ShiftService.clockOut()` already mutates accrued wages inside a transaction, so avoid duplicate accrual paths.
6. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter, including payment authority, idempotency, tab closure, tips, realtime and retry-after-timeout.
7. Strengthen Admin financial mutation coverage and optimistic pending-queue correctness.
8. After these P0s, execute tenant/state/realtime matrices, then production-ops/load/red-team waves.

## Duplicate-work prohibition

Do not recreate merged backend hardening work or the superseded POS/inventory/kiosk PRs. #140 and #142 are intentionally closed; #145 and #144 are the current replacements on current main.

## Continuation rule

When CI is running, continue independent research/audit work rather than waiting. When CI completes, return to its exact-head gate before merge. When a work package reaches verified main, immediately advance to the next P0. A fresh conversation must resume from this file and current GitHub state, not from memory of an earlier session.

## Last reconciled references

- Backend main: `a0876b2f61d5bc73acb1a1d76368e019d079fe82`
- Inventory merge: `a0876b2f61d5bc73acb1a1d76368e019d079fe82` / PR #141
- Kiosk PR #144 head: `4fca0d65a416b85cefa22ec4b15256b5a6cff25d`
- Kiosk exact-head run: `33776303876` — in progress
- POS PR #145 head: `c01ab35692f7cf237387ecf00d5454ad748a2c57`
- POS exact-head run: `33776598303` — in progress
- Frontend PR #78 merge: `7cf85fa1942c277f5b2de4578e41d75cf81b20a5`

## Completion record

This loop remains active until #144 and #145 have reliable exact-head CI and verified merges are on current `main`. Then advance immediately to POS semantic closure, dine-in payment authority, `updateAccruedWages` trace, Admin financial mutation coverage and production hardening.
