# AZAMAN Active Engineering Loop

**Purpose:** persistent continuation state for long-running engineering work. A new session MUST resume from this file when it is non-empty/active rather than restarting or asking the user to restate the task.

## Operating contract

1. Read `START_HERE.md`, `ROADMAP.md`, `CURRENT_STATE.md`, then this file.
2. Reconcile every referenced repo/PR/branch/SHA against GitHub before changing code.
3. Before creating a branch or PR, search existing branches, open PRs, recent merges and equivalent implementations/tests.
4. Execute the entire active work package: research → trace producers/consumers → inspect schema/authorization/state machine → implement → test → exact-head CI → diff audit → merge → verify `main` → update this file and `CURRENT_STATE.md`.
5. Do not stop merely because a defect was found, a patch was written, a local test passed, or a PR was opened.
6. If CI fails, diagnose and fix it; do not weaken tests/type checking to obtain green CI.
7. After a verified completion, immediately select the next unchecked P0 from `ROADMAP.md` unless genuinely blocked.
8. If blocked by missing access, an external dependency, or a decision that cannot safely be inferred, record the exact blocker and the smallest required user action. Otherwise continue autonomously.
9. Never duplicate completed work. If an existing implementation is partial, continue/consolidate it instead of creating a competing implementation.

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Verified completed in this continuation

- Backend PR #131 payroll Serializable retry verified green and merged to main as `e81e32d83262a27c721ac51de8786b650f6be433`.
- Backend PR #132 shift generic-status mutation boundary verified with exact-head Azaman Test Suite run `33724745236` / run #560 and merged as `ea52b6b82cc6c703bcc66b2dcf4aa7a34681ea8b`.
- Backend PR #135 dine-in finalization/item-price/payment-idempotency hardening merged as `fc24fcc61800e86cad9657b91b59c8d24e93a8ef`.
- Backend PR #136 KYB gate fail-closed hardening merged.
- Backend PR #137 canonical Business OS Finance runtime/route repair merged; current Backend main is `924807b3742f30f929479d46dba96d9660b61f2d`.
- Planning PR #25 established persistent `ACTIVE_LOOP.md` + `EXECUTION_LEDGER.json` continuation state.

### Current active work package: Business OS financial/operational mutation correctness

#### POS — current replacement PR
- Former PR #138 is closed and superseded because its branch accumulated diagnostic/CI history.
- PR #140 (`fix(pos): atomic Business OS POS settlement (current main)`) is the current POS branch, `fix/business-os-pos-atomicity-v2`, head `c1e4186caac0161fea41f253fd96703bbb3980e8`.
- Current implementation server-derives catalog prices, validates integer quantities, handles CASH/AZM/SPLIT, conditionally debits AZM, recovers idempotency races, creates the order and ledger atomically, and now persists authoritative `BusinessOrderItem` rows in the same transaction.
- Do not merge until an exact-head green `Azaman Test Suite` exists. The known PR run #584 targeted older head `10b8c326...` and failed before normal job steps; integration-written later commits have not produced a new exact-head PR run.
- Remaining POS contract audit: tax authority, ledger units, customer/payment identity, inventory effects, and replay semantics.

#### Inventory restock
- PR #139 remains open at head `f3c945204333ccd6074fcfd6571e7336386f249e`.
- Restock now mutates stock and records signed expense plus GHS-equivalent ledger amount in one transaction.
- Do not merge until exact-head CI is green. Recent failures ended before executable job steps were recorded.
- Remaining audit: retry/idempotency semantics and ledger producer conventions.

### CI blocker

Current integration-driven writes are producing stale/missing pull-request workflow runs for newly written heads, and some recent runs fail with a job object containing no executable steps. The canonical workflow itself remains known-good: Backend run `33732480681` executed all setup, tests and DB recovery stages successfully. The release gate must not be weakened to bypass this discrepancy.

### Next exact actions

1. Resolve/obtain a reliable exact-head CI execution path for PR #140 and #139 without changing release standards.
2. Once green, perform final diff audit, merge, verify Backend `main`, then reconcile PR #139 against the new main head and gate it independently.
3. Finish kiosk capability hardening: token expiry/scope tests, employee revalidation, location-binding decision, brute-force/rate-limit review, exact-head CI.
4. Audit POS tax/customer/inventory/ledger contracts against existing producers/consumers before calling POS complete.
5. Continue the earlier shift route authorization package: rotation permission parity, attendance/EWA route-vs-service matrix, and `updateAccruedWages` history/consumer trace.
6. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter, including concurrent payment, idempotency, tab closure, tip authority, realtime and client convergence.
7. Then proceed to Admin financial mutation coverage and the remaining tenant/state/realtime/production-ops waves.

## Duplicate-work prohibition

Do not recreate merged PR #131/#132/#135/#136/#137 or the existing POS/inventory implementations. Reconcile current GitHub state before creating adjacent work.

## Continuation rule

When CI is running, continue independent research/audit work rather than waiting. When CI completes, return to its exact-head gate before merge. When a work package reaches verified main, immediately advance to the next P0. A fresh conversation must resume from this file and current GitHub state, not from memory of a previous conversation.

## Last reconciled references

- Backend main: `924807b3742f30f929479d46dba96d9660b61f2d`
- POS replacement PR #140 head: `c1e4186caac0161fea41f253fd96703bbb3980e8`
- Inventory PR #139 head: `f3c945204333ccd6074fcfd6571e7336386f249e`
- Known-good Backend full CI run: `33732480681`
- Planning reconciliation PR: #28

## Completion record

This loop remains active until the current Backend mutation batch has reliable exact-head CI and the verified merges are on `main`. Then advance through kiosk/shift authorization and the dine-in payment authority loop without restarting from scratch.
