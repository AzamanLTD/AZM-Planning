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

**Primary work package:** Backend Business OS shift mutation/state boundary, followed immediately by dine-in payment contract hardening.

### Verified completed during this continuation

- Backend PR #131 payroll Serializable P2034 retry verified green and merged to main as `e81e32d83262a27c721ac51de8786b650f6be433`.
- Backend PR #132 shift generic-status mutation boundary verified with exact-head Azaman Test Suite run `33724745236` / run #560 and merged as `ea52b6b82cc6c703bcc66b2dcf4aa7a34681ea8b`.
- Main verification confirms `ShiftService.updateShift()` rejects `updates.status` before database access and the generic allowlist no longer contains `status`.
- Planning PR #25 merged, establishing this persistent `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` mechanism.
- Flutter PR #78 is open for a confirmed client truthfulness defect: `DineInTabNotifier.payTab()` now rethrows payment failures instead of swallowing them, so Confirm & Pay cannot show a false success. Exact-head Flutter Quality run `33725099978` is still running; analyze has passed and test/coverage is in progress.

### Important audit findings already established

- `POST /shifts/rotation` lacks `requirePermission('shifts.create')` while normal shift creation requires it. This remains unresolved and is the next backend route-boundary patch.
- Attendance route handlers rely on service-level actor/business authorization; do not add middleware blindly until the actor matrix is reconciled.
- `EmployeeService.updateAccruedWages(employeeId, hoursWorked)` appears to have no in-repo consumers beyond its definition; current canonical `ShiftService.clockOut()` performs employee wage/hour accounting transactionally. Before removal, re-check repository-wide history/consumers and preserve compatibility if external callers cannot be ruled out.
- Dine-in `confirmAndPay` currently checks tab state before the payment transaction and can create an invoice when none exists. The cross-repo audit must establish idempotency/unique invoice behavior, concurrent confirm-and-pay behavior, tab closure race semantics, tip authority, and realtime reconciliation before implementation.
- Flutter currently swallowed `/dine-in/tabs/:tabId/pay` failures, which was fixed on PR #78; this is a client-side contract truthfulness fix, not a substitute for backend idempotency.
- `FinanceV2.jsx` in Business Portal is a one-line re-export of `Finance.jsx`; it is not independently authoritative and should only be removed after route/import reachability is proven.

### Remaining within the current backend shift loop

1. Backend PR #133 adds rotation permission parity, protects management attendance mutations with `shifts.update`, and applies `ewa.manage` to the business EWA management endpoints. It also carries owner/admin authority into both request-context stores used by the service layer. Focused local Jest verification passes (12 tests); do not merge until exact-head CI and a final diff audit pass.
2. Audit attendance endpoint authorization matrix (`clock-in`, `clock-out`, `no-show`) against service semantics and kiosk exceptions.
3. Trace `updateAccruedWages` consumers across code/history before deciding removal/restriction.
4. Exact-head CI, final diff audit, merge and main verification for the route/authorization batch.

### Next P0 loop: dine-in cross-repo payment authority

Backend → Business Portal → Flutter. Start with `confirmAndPay` and trace:

- tab ownership/tenant scope;
- item/product price authority;
- invoice creation and uniqueness/idempotency;
- payment/debit ledger/balance effects;
- transaction boundaries;
- concurrent confirm-and-pay requests;
- already-paid behavior;
- tip authority and replay;
- tab status transition/closure;
- invoice-to-tab linkage;
- websocket/realtime events;
- client success/failure semantics;
- retry/offline recovery and missed-event reconciliation.

Implement only after the complete producer/consumer trace is understood; add concurrency/idempotency regression coverage; exact-head CI; merge; verify main.

## Duplicate-work prohibition

Do not recreate PR #131, #132, or #78. Reconcile their current GitHub state before touching adjacent code. If another branch/PR contains equivalent work, consolidate rather than duplicate.

## Continuation rule

When a CI job is running, continue independent research/audit work rather than waiting. When CI completes, return to its exact-head gate before merge. When a work package reaches verified main, immediately advance to the next P0. A fresh conversation must resume from this file and current GitHub state, not from memory of a previous conversation.

## Last verified references

- Backend main after PR #132: `ea52b6b82cc6c703bcc66b2dcf4aa7a34681ea8b`
- Planning main after PR #25: `fbb3ac4cb83f7859e9bdc6a174c71dbd78272532`
- Flutter PR #78 head: `ce5f29b4d6f52053088693df8c81aa544f5a1ad8`
- Flutter PR #78 CI: `33725099978` (in progress at last check)

## Completion record

This loop remains active until the unresolved shift route/authorization batch is on main and verified. Then immediately advance to the dine-in payment authority loop. Do not mark work complete merely because a PR exists.
