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

- Backend PR #220 atomically initializes missing `OrderTracking` rows and is merged.
- Backend PR #221 serializes tracking mutations per order, moves event timestamps inside the serialization boundary, initializes ETA writes safely, and validates tracking mutation payloads; merged/verified exact-head.
- Backend PR #223 modernizes backend Actions checkout/setup-node major versions; merged after exact-head Actions run #847 passed.
- Business Portal latest main includes storefront responsive inheritance, keyboard tile movement, responsive preview viewports, custom-HTML preview sanitization, legacy tile collision safety, socket/session lifecycle cleanup, multi-page tree ownership, category policy aliases, deterministic Studio migration IDs, and API error/multipart contract hardening.
- Flutter latest main uses canonical USDC/GHS retail-rate provenance for live FX, home-market display/history, rate alerts, and Susu display.
- Admin Portal latest main includes concurrency-safe withdrawal optimistic mutations plus refresh interaction hardening.
- Backend PR #222 (`fix(finance): bind provider references to one transaction`) is OPEN and intentionally NOT merged; its setup/schema/Prisma stages passed but the test stage failed. Do not promote it until the actual test failure is diagnosed and exact-head CI is green.
- Flutter PR #88 (`chore(ci): update frontend checkout action`) is OPEN; verify exact-head quality/Android workflows before merge.
- Business Portal and Admin Portal currently have no open PRs.

### Current backend baseline

`b2d74f6bc5b4730ed998e4c42bf1efaf6a7032da`

### Active P0 work

#### 1. Resolve provider-attempt correlation failure — PR #222

Branch: `fix/provider-attempt-reference-integrity`  
Head: `d3a5a43e60c895eeda0171b6c702a45b279db993`

- Inspect the failed test output through every available GitHub Actions/check interface before changing behavior.
- Preserve the intended invariant: one `(provider, providerReference)` must remain bound to exactly one canonical `TransactionHistory` row.
- Keep the conditional `ON CONFLICT` behavior and deterministic collision failure if the implementation remains correct; fix only the proven failure.
- Exact-head full CI and database recovery drill are mandatory before merge.

#### 2. Complete the canonical tax/commerce authority wave

After #222 is resolved, resume the highest-value unfinished roadmap items:

- audit POS/invoice tax authority across `BusinessInvoice`, `BusinessInvoiceTaxLine`, `BusinessTaxPreset` and all producers/consumers;
- verify location/global product authority and tenant boundaries across POS and dine-in;
- audit order/invoice payment, settlement, refund/void and duplicate-request behavior;
- inspect reservation/booking payment and capacity races.

#### 3. Cross-client dine-in lifecycle proof

Trace and test the complete contract:

`Flutter → Backend → Business Portal → Admin visibility`

Focus on FINALIZED/CLOSED semantics, tips, paid replay, reconnect/background ordering, multi-tab races, timeout recovery, authoritative refresh, and socket/poll convergence.

#### 4. Financial/control-plane integrity

Continue tenant/state/realtime waves for withdrawals, escrow, trades, wallet mutation, payroll/EWA and admin approvals/releases. Every read → decision → write path must be protected by conditional writes, transactions, unique constraints or deterministic state claims as appropriate.

#### 5. Production readiness

After correctness gates are substantially complete, prove deployment/configuration separation, secrets handling/rotation, migration rollback discipline, observability, worker recovery, anomaly detection, load behavior and adversarial scenarios required by `ROADMAP.md`.

## Parallel execution rule

When a CI run is active, perform independent audits, contract tracing, or repository reconciliation instead of waiting idle. Do not parallelize two changes that modify the same logical authority boundary without first selecting one canonical implementation.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is considered complete until its exact PR head has passed the relevant full test/type/build gate and all required database recovery evidence, followed by verification on `main`.

## Planning synchronization

After every verified merge:

1. reconcile current repo main SHAs;
2. update `CURRENT_STATE.md` with evidence and residual risk;
3. update `ACTIVE_LOOP.md` to the next P0;
4. update `EXECUTION_LEDGER.json` with the verified implementation/CI evidence;
5. continue immediately to the next unchecked P0.

## Duplicate-work prohibition

Do not resurrect superseded PRs/branches when current `main` already contains the required work. In particular, do not revive old POS, kiosk, dine-in, storefront or Planning attempts that were explicitly superseded.
