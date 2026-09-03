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
- Planning persistent continuation files are established and remain the canonical continuation mechanism.

### Current active work package: Business OS financial/operational mutation correctness

#### POS — current replacement PR #140
- Former PR #138 is closed and superseded.
- PR #140 (`fix(pos): atomic Business OS POS settlement (current main)`) is open on `fix/business-os-pos-atomicity-v2` at head `b96bfa64491fb0e8790dcc6c4b1720237e17976c`.
- Implementation server-derives catalog prices, validates integer quantities, supports CASH/AZM/SPLIT, conditionally debits AZM, persists `BusinessOrderItem` rows, and commits order/line-items/ledger atomically with Serializable retry.
- Idempotent replay resolves before catalog validation so safe offline retries survive catalog changes while tenant isolation remains enforced.
- Legacy producer tracing confirms the existing `/pos/order` uses the same 2.5% POS tax behavior and `customerId || authenticatedUserId` default, so those are not unexplained contract deviations.
- Exact-head full CI remains unproven; no fresh run is attached to the current head. Do not merge without a green exact-head `Azaman Test Suite`.

#### Inventory — current replacement PR #141
- Older duplicate PR #139 is **closed and not merged**; PR #141 is now the sole current inventory path.
- PR #141 (`fix(inventory): atomic restaurant restock on current main`) is open on `fix/business-os-inventory-restock-atomicity-v2` at head `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`.
- Stock plus signed SUPPLIES expense are committed transactionally with business/quantity/cost validation and ledger-failure coverage.
- Temporary diagnostic workflow artifacts were removed.
- Exact-head runs `33746032891`, `33746106832`, `33746223780`, `33746364784` (rerun attempt 2) failed before executable steps; do not merge on that evidence.

#### Kiosk — current PR #142
- PR #142 (`fix(kiosk): enforce scoped clock capability and location binding`) is open on `fix/business-os-kiosk-capability` at head `40c78279c87e818071f0b9317149e96904ddc3eb`.
- Capability signing/verification is isolated in `services/businessOS/kioskCapability.js`.
- Clock actions revalidate active employee and target shift against capability tenant/employee/user binding; location-bound capabilities must match the shift location.
- PIN authentication validates a supplied location belongs to the business.
- Focused tests cover expiry, non-kiosk scope, tenant/employee/location binding and unbound-location behavior.
- Exact-head CI run `33746899789` (#608) failed before executable steps; this matches the cross-PR runner/job-startup failure pattern. Do not merge without exact-head green evidence.

### CI blocker / evidence rule

The canonical Backend workflow remains unchanged. Known-good PR #137 run `33732480681` executed setup, tests and the DB recovery drill successfully. Current fresh PR heads can fail before executable steps are exposed; earlier equivalent attempts also reached the real test stage and failed. Treat this as an evidence/reliability blocker, not permission to weaken the release gate.

### Next exact actions

1. Obtain reliable exact-head CI for #140/#141/#142 without changing release standards.
2. Complete POS inventory/ledger/location/replay semantics audit; merge only after exact-head green evidence and final diff audit.
3. Strengthen kiosk abuse-resistance/rate-limit review, then exact-head CI.
4. Trace every `updateAccruedWages()` consumer/history before any removal or restriction.
5. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter.
6. Add Admin financial mutation tests and optimistic queue correctness.
7. Complete tenant/state/realtime matrices, then production-ops/load/red-team waves.

## Duplicate-work prohibition

Do not recreate merged PR #131/#132/#135/#136/#137 or the existing POS/inventory/kiosk implementations. Reconcile current GitHub state before creating adjacent work.

## Continuation rule

When CI is running, continue independent research/audit work rather than waiting. When CI completes, return to its exact-head gate before merge. When a work package reaches verified main, immediately advance to the next P0. A fresh conversation must resume from this file and current GitHub state, not from memory of a previous conversation.

## Last reconciled references

- Backend main: `924807b3742f30f929479d46bda96d9660b61f2d`
- POS PR #140 head: `b96bfa64491fb0e8790dcc6c4b1720237e17976c`
- Inventory PR #141 head: `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`
- Kiosk PR #142 head: `40c78279c87e818071f0b9317149e96904ddc3eb`
- Known-good Backend full CI run: `33732480681`
- Latest inventory CI run/rerun: `33746364784`, attempt 2 zero-step failure
- Latest kiosk CI run: `33746899789`, zero-step failure

## Completion record

This loop remains active until the current Backend mutation batch has reliable exact-head CI and verified merges are on `main`. Then advance through shift authorization, dine-in payment authority, Admin coverage and production hardening without restarting from scratch.
