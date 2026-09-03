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
- Backend PR #137 canonical Business OS Finance runtime/route repair merged; current Backend main remains `924807b3742f30f929479d46bda96d9660b61f2d`.
- Planning persistent continuation files (`ACTIVE_LOOP.md` + `EXECUTION_LEDGER.json`) are established and remain the canonical continuation mechanism.

### Current active work package: Business OS financial/operational mutation correctness

#### POS — current replacement PR
- Former PR #138 is closed and superseded because its branch accumulated stale-base/diagnostic history.
- PR #140 (`fix(pos): atomic Business OS POS settlement (current main)`) is current, branch `fix/business-os-pos-atomicity-v2`.
- Current head: `b96bfa64491fb0e8790dcc6c4b1720237e17976c`.
- Current implementation server-derives business catalog prices, validates integer quantities, supports CASH/AZM/SPLIT, conditionally debits AZM, persists authoritative `BusinessOrderItem` rows, and commits order/line-items/ledger atomically with Serializable retry.
- Idempotent replay is deliberately resolved before catalog validation, preserving safe offline replay after catalog changes while retaining cross-business key isolation.
- Exact-head CI is **not verified**: no workflow run is attached to the current head `b96bfa...`. Earlier attempts produced genuine test-stage failures and later runner/job-startup failures. Do not merge without an exact-head green full `Azaman Test Suite` run.
- Remaining POS contract audit: tax authority, ledger currency/unit semantics, customer/payment identity, inventory effects, location/table integrity, and replay response contract.

#### Inventory restock — current replacement PR
- Older duplicate PR #139 remains open for historical reconciliation but is superseded by current PR #141 and must not be merged alongside it.
- PR #141 (`fix(inventory): atomic restaurant restock on current main`) is current, branch `fix/business-os-inventory-restock-atomicity-v2`.
- Current head: `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`.
- Current implementation scopes inventory to business, validates quantity/cost, and commits stock plus signed SUPPLIES expense in one transaction. Focused coverage includes ledger failure propagation.
- Temporary/accidental diagnostic workflow artifacts were removed; current PR diff is the intended four files.
- Exact-head CI remains **not verified**. Runs `33746032891` (#604), `33746106832` (#605), `33746223780` (#606), and `33746364784` (#607) failed before executable job steps were exposed. Run #607 was explicitly rerun and again failed at job startup. Do not merge without a current exact-head green full run.

### CI blocker / evidence rule

The canonical backend workflow remains unchanged. Known-good PR #137 run `33732480681` executed setup, tests and the DB recovery drill successfully. Current fresh PR runs can fail before executable steps are available through the GitHub integration, while earlier equivalent attempts also reached the real test stage and failed. This is a release-evidence blocker, not permission to weaken the gate. Preserve the full workflow and obtain reliable exact-head execution evidence.

### Next exact actions

1. Reconcile the latest CI attempt(s) for #141 and #140; obtain reliable exact-head green evidence without weakening the release workflow.
2. Close/supersede duplicate PR #139 deliberately once #141 remains the sole inventory implementation path; then gate #141 independently against current main.
3. Complete POS contract audit (tax/customer/inventory/ledger/location/replay) before calling POS release-ready.
4. Finish kiosk capability hardening: expiry/scope tests, rate-limit review, location binding, exact-head CI.
5. Trace every `updateAccruedWages()` consumer/history before any removal or restriction.
6. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter, including concurrent payment, idempotency, tab closure, tip authority, realtime and client convergence.
7. Continue Admin financial mutation coverage, then tenant/state/realtime/production-ops waves.

## Duplicate-work prohibition

Do not recreate merged PR #131/#132/#135/#136/#137 or the existing POS/inventory implementations. Reconcile current GitHub state before creating adjacent work.

## Continuation rule

When CI is running, continue independent research/audit work rather than waiting. When CI completes, return to its exact-head gate before merge. When a work package reaches verified main, immediately advance to the next P0. A fresh conversation must resume from this file and current GitHub state, not from memory of a previous conversation.

## Last reconciled references

- Backend main: `924807b3742f30f929479d46bda96d9660b61f2d`
- POS replacement PR #140 head: `b96bfa64491fb0e8790dcc6c4b1720237e17976c`
- Inventory replacement PR #141 head: `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`
- Known-good Backend full CI run: `33732480681`
- Latest inventory CI run / rerun: `33746364784` (run attempt 2 still zero-step/startup failure)
- Planning reconciliation branch: `reconcile/2026-09-03-pos-inventory-ci-state`

## Completion record

This loop remains active until the current Backend mutation batch has reliable exact-head CI and the verified merges are on `main`. Then advance through kiosk/shift authorization and the dine-in payment authority loop without restarting from scratch.
