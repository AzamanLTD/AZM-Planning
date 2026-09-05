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

- Backend main: `7afbe1a7318402519aafed6142fdb7364f9df86d`; security remediation PR #230, retail catalog PR #231, and dine-in recovery notification PR #232 are merged and verified.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9`; durable dine-in payment recovery PR #89 is merged.
- Business Portal main: `73b9763aa90c258ab496c2c355b78384bbe53568`; Studio Wave A and Wave B are merged. Wave B PR #85 exact-head CI run #239 passed smoke, 46/46 test files / 146 tests, and production build.
- Admin Portal main: latest verified withdrawal optimistic-concurrency hardening remains on main.
- Planning main: reconciliation is in progress for these newly verified merges.

### Active P0 work

#### 1. Cross-client dine-in lifecycle executable proof

The underlying authority is now hardened through `OPEN → FINALIZED → CLOSED`, durable payment replay, concurrent payment recovery, and business-owner notification on recovered payment. The next package is executable proof across Flutter → Backend → Business Portal → Admin: finalize/invoice/payment/CLOSED, replay/races/tips, lost-response/reconnect/background recovery, multi-tab races, and authoritative read-model convergence.

#### 2. Canonical business-invoice idempotency/replay audit

Map remaining runtime `createInvoice` callers outside POS using repository-tree/file inspection where GitHub search is unavailable. Preserve tenant/customer binding, canonical invoice math, and replay semantics; change only concrete gaps.

#### 3. Storefront Studio continuation

Wave B is verified and merged. Do not broaden behavior until a concrete gap is identified. Next Studio target is Wave C device emulation: actual 412x892 phone frame, scrollable content, and expanded-width breakpoint behavior while preserving Wave A/B contracts.

#### 4. Financial/control-plane integrity

Continue tenant/state/realtime waves across withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, payroll/EWA, refunds/voids, reservations, approvals and operational/load evidence.

#### 5. Production readiness and adversarial verification

Prove deployment/configuration separation, secret handling/rotation, migration rollback discipline, observability, worker recovery, anomaly alerts, concurrency/load behavior and release rehearsal.

## Parallel execution rule

When a CI run is active, perform independent audits, contract tracing, or repository reconciliation instead of waiting idle. Do not parallelize two changes that modify the same logical authority boundary without first selecting one canonical implementation.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is considered complete until its exact PR head has passed the relevant full test/type/build gate and all required database recovery evidence, followed by verification on `main`.

Studio UI changes must pass the Business Portal smoke/test/build gate and targeted Studio contract tests before merge.

## Planning synchronization

After every verified merge:

1. reconcile current repo main SHAs;
2. update `CURRENT_STATE.md` with evidence and residual risk;
3. update `ACTIVE_LOOP.md` to the next P0;
4. update `EXECUTION_LEDGER.json` with verified implementation/CI evidence;
5. continue immediately to the next unchecked P0.

## Duplicate-work prohibition

Do not resurrect superseded PRs/branches when current `main` already contains the required work. In particular, do not revive old POS, kiosk, dine-in, storefront or Planning attempts that were explicitly superseded.