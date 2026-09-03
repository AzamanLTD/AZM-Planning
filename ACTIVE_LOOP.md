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

- Backend PR #131 payroll Serializable retry verified green and merged to main as `e81e32d83262a27c721ac51de8786b650f6be433`.
- Backend PR #132 shift generic-status mutation boundary verified with exact-head Azaman Test Suite run `33724745236` / run #560 and merged as `ea52b6b82cc6c703bcc66b2dcf4aa7a34681ea8b`.
- Backend PR #135 dine-in finalization/item-price/payment-idempotency hardening merged as `fc24fcc61800e86cad9657b91b59c8d24e93a8ef`.
- Backend PR #136 KYB gate fail-closed hardening merged.
- Backend PR #137 canonical Business OS Finance runtime/route repair merged; current Backend main remains `924807b3742f30f929479d46bda96d9660b61f2d`.
- Frontend PR #78 dine-in payment failure propagation verified with exact-head Flutter Quality run `33725099978` / run #314 and merged to main as `7cf85fa1942c277f5b2de4578e41d75cf81b20a5`.
- Planning persistent continuation files are verified present on Planning main.

### Current active work package: Business OS financial/operational mutation correctness

#### POS — current replacement PR #140
- Former PR #138 is closed and superseded.
- PR #140 is open on `fix/business-os-pos-atomicity-v2` at head `6a2947447aec60528033d1ef0a7416bebf5b2b05`; GitHub currently reports `mergeable=false`, `merged=false`.
- Implementation server-derives catalog prices, validates integer quantities, supports CASH/AZM/SPLIT, conditionally debits AZM, persists `BusinessOrderItem` rows, and commits order/line-items/ledger atomically with Serializable retry.
- Idempotent replay resolves before catalog validation so safe offline retries survive catalog changes while tenant isolation remains enforced.
- Legacy producer tracing confirms the existing `/pos/order` used a 2.5% POS tax and `customerId || authenticatedUserId` fallback. Treat these as compatibility findings, not final accounting authority.
- Remaining contract risks: hardcoded 2.5% tax versus `BusinessTaxPreset`; inventory decrement; location/table integrity; cash customer identity; and replay payload binding.
- Exact-head green CI is unproven. No merge until a full current-head `Azaman Test Suite` succeeds.

#### Inventory — current PR #141
- Older PR #139 is closed/superseded and was never merged.
- PR #141 is open on `fix/business-os-inventory-restock-atomicity-v2` at current head `fc5b1df14a99a4895860649b850f7c575265b42e`; GitHub reports `mergeable=true`, `merged=false`.
- The current PR contains only four intended files: the canonical inventory service/route, its focused test, and route registration. The unmerged POS router dependency was removed.
- Stock plus signed SUPPLIES expense execute in one transaction; business scoping, quantity/cost validation and ledger-failure propagation are covered.
- Earlier runs reached the real test stage and failed. The newest exact-head run `33749730364` failed in roughly three seconds with no executable steps or log blob, and an immediate rerun again produced no steps. Temporary diagnostic workflows were removed after themselves failing at runner startup.
- Do not merge until a current exact-head full test run executes and passes.

#### Kiosk — current PR #142
- PR #142 is open on `fix/business-os-kiosk-capability` at head `40c78279c87e818071f0b9317149e96904ddc3eb`; GitHub reports `mergeable=true`, `merged=false`.
- Capability signing/verification is isolated; clock-in/out enforce tenant/employee/user/shift binding and location binding; PIN auth validates supplied location belongs to the business.
- Focused tests cover expiry, non-kiosk scope, tenant, employee and location binding.
- Exact-head run `33746899789` failed before executable steps were exposed. Do not merge without exact-head green evidence.

### CI / release-gate evidence

The canonical Backend workflow remains unchanged. Known-good PR #137 run `33732480681` executed setup, tests and the DB recovery drill successfully. Current active PRs exhibit both historical test-stage failures and current zero-step startup failures. Preserve the release gate; never weaken tests to manufacture green status.

### Exact next actions

1. Continue obtaining reliable current-head CI for #140/#141/#142 without changing the release standard.
2. While CI is unreliable, complete the POS contract audit, especially tax authority, inventory decrement, location/table scope and idempotency payload binding.
3. Complete kiosk abuse-resistance/rate-limit review.
4. Trace every `updateAccruedWages()` consumer/history before any removal or restriction.
5. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter.
6. Add Admin financial mutation coverage and optimistic pending-queue correctness.
7. Complete tenant/state/realtime matrices, then production-ops/load/red-team waves.

## Duplicate-work prohibition

Do not recreate merged PR #131/#132/#135/#136/#137/#78 or existing POS/inventory/kiosk implementations. Reconcile current GitHub state before adjacent work.

## Continuation rule

When CI is running, continue independent research/audit work rather than waiting. When CI completes, return to its exact-head gate before merge. When a work package reaches verified main, immediately advance to the next P0. A fresh conversation must resume from this file and current GitHub state, not from memory of an earlier session.

## Last reconciled references

- Backend main: `924807b3742f30f929479d46bda96d9660b61f2d`
- POS PR #140 head: `6a2947447aec60528033d1ef0a7416bebf5b2b05`
- Inventory PR #141 head: `fc5b1df14a99a4895860649b850f7c575265b42e`
- Latest Inventory exact-head run: `33749730364` — zero-step/job-startup failure
- Kiosk PR #142 head: `40c78279c87e818071f0b9317149e96904ddc3eb`
- Latest known Kiosk exact-head run: `33746899789` — zero-step/job-startup failure
- Frontend PR #78 merge: `7cf85fa1942c277f5b2de4578e41d75cf81b20a5`
- Known-good Backend full CI: `33732480681`

## Completion record

This loop remains active until the current Backend mutation batch has reliable exact-head CI and verified merges are on `main`. Then advance through shift authorization, dine-in payment authority, Admin coverage and production hardening without restarting from scratch.
