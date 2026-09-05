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

### Verified current GitHub state (2026-09-05 UTC)

- Backend main: `d230aa0d5e865f3b39b828f2101a01afb20ac762`; security remediation PR #230 and storefront catalog PR #231 are merged and verified.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9`; durable dine-in payment recovery PR #89 is merged.
- Business Portal main: `563bfecf8187e93a6b5990505ecfee64efd81a34`; Studio Wave A retail preview completion PR #84 is merged after exact-head CI passed smoke tests, full tests, and build.
- Admin Portal main: latest verified withdrawal optimistic-concurrency hardening remains on main.
- Planning main: reconciled through this update.

### Active P0 work

#### 1. Dine-in payment recovery notification proof

PR #232 is the current active backend slice. The durable recovery path now emits the same `DINE_IN_TAB_PAID` business-owner notification as the normal success path. Exact-head CI must pass before merge; then verify backend main.

#### 2. Cross-client dine-in lifecycle executable proof

Convert the hardened authority into end-to-end proof across Flutter → Backend → Business Portal → Admin. Cover finalize/invoice/payment/CLOSED, idempotent replay, duplicate/concurrent payment attempts, tips, lost-response timeout/reconnect/background recovery, multi-tab races, and authoritative read-model convergence.

#### 3. Canonical business-invoice idempotency/replay audit

Map all runtime `createInvoice` callers outside POS, preserve tenant/customer binding and canonical invoice math, and harden durable replay where the current service contract is still only uniqueness-based.

#### 4. Storefront Studio Wave B

Wave A is verified and merged. Reuse its immutable measured token source and semantic/runtime adapter. Wave B may now introduce pointer capture, magnetic snap, connected-group fusion, and deterministic settle animation, but must preserve the semantic-tree model and avoid any AI/free-canvas/localStorage behavior.

#### 5. Financial/control-plane integrity

Continue tenant/state/realtime waves across withdrawals, wallet, escrow, trades, payroll/EWA, refunds/voids, reservations and admin approvals.

#### 6. Production readiness and adversarial verification

Prove deployment/configuration separation, secret handling/rotation, migration rollback discipline, observability, worker recovery, reconciliation/anomaly alerts, concurrency/load behavior, and release rehearsal.

## Parallel execution rule

When a CI run is active, perform independent audits, contract tracing, or repository reconciliation instead of waiting idle. Do not parallelize two changes that modify the same logical authority boundary without first selecting one canonical implementation.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is considered complete until its exact PR head has passed the relevant full test/type/build gate and all required database recovery evidence, followed by verification on `main`.

Studio UI changes must pass the Business Portal smoke/test/build gate and targeted Storefront Studio contract tests before merge.

## Planning synchronization

After every verified merge:

1. reconcile current repo main SHAs;
2. update `CURRENT_STATE.md` with evidence and residual risk;
3. update `ACTIVE_LOOP.md` to the next P0;
4. update `EXECUTION_LEDGER.json` with the verified implementation/CI evidence;
5. continue immediately to the next unchecked P0.

## Duplicate-work prohibition

Do not resurrect superseded PRs/branches when current `main` already contains the required work. In particular, do not revive old POS, kiosk, dine-in, storefront or Planning attempts that were explicitly superseded.