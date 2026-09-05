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
- Backend PR #224 serializes BusinessTaxPreset default authority with a transaction-scoped advisory lock and database uniqueness migration; merged/verified.
- Backend PR #222 binds external provider references to exactly one canonical `TransactionHistory`; exact-head Actions run #855 passed tests and database recovery, then the PR was merged at `f8c433be22fa29dd7e0ce7134257020aa56736a8`.
- Backend PR #225 serializes dine-in item writes against tab finalization; its initial test failure was corrected at the adapter-test boundary, exact-head run #856 passed, then it was merged at `bb542f68404e095ac0438be0747d73ebeb23cc05`.
- Business Portal latest main includes storefront responsive inheritance, keyboard tile movement, responsive preview viewports, custom-HTML preview sanitization, legacy tile collision safety, socket/session lifecycle cleanup, multi-page tree ownership, category policy aliases, deterministic Studio migration IDs, and API error/multipart contract hardening.
- Flutter latest main uses canonical USDC/GHS retail-rate provenance for live FX, home-market display/history, rate alerts, and Susu display; PR #89 additionally converges ambiguous dine-in payment responses from durable CLOSED state.
- Admin Portal latest main includes concurrency-safe withdrawal optimistic mutations plus refresh interaction hardening.
- Business Portal and Admin Portal currently have no open PRs.

### Current backend baseline

`f8c433be22fa29dd7e0ce7134257020aa56736a8`

### Active P0 work

#### 1. Cross-client dine-in lifecycle proof

Trace and prove the complete contract:

`Flutter → Backend → Business Portal → Admin visibility`

Cover the authoritative path `OPEN → FINALIZED → invoice DRAFT/SENT → payment → CLOSED`, including:

- duplicate/concurrent item mutations versus finalization;
- invoice creation and payment idempotency/replay;
- tips and fee coverage economics;
- lost response / timeout / reconnect / background ordering;
- multi-tab and repeated payment races;
- Business Portal and Admin read models observing the same invoice/payment truth.

The item/finalization race and Flutter ambiguous-payment recovery have already been hardened. The remaining work is contract proof and any additional invariant uncovered by tracing.

#### 2. POS/invoice tax-authority producer/consumer audit

Now that `BusinessTaxPreset` default authority is transactionally serialized and database-unique, trace every tax producer/consumer across:

- `BusinessTaxPreset`;
- `BusinessInvoice` / `BusinessInvoiceTaxLine`;
- POS checkout and product/location flows;
- dine-in invoice creation;
- legacy Business OS finance routes.

Do not introduce a second tax authority. Preserve the existing contract where omitted `taxLines` selects the oldest default preset and explicit `[]` means tax-free unless the producer/consumer audit proves otherwise.

#### 3. Canonical business-invoice creation idempotency/replay

After caller mapping and tax-authority tracing, harden `createInvoice` so a client-supplied idempotency key is a durable replay boundary rather than merely a unique database field. Preserve business/customer tenant binding and avoid duplicating invoice math or creating a competing service authority.

#### 4. Financial/control-plane integrity

Continue tenant/state/realtime waves for withdrawals, wallet mutation, escrow, trades, payroll/EWA, refunds/voids, reservations and admin approvals. Every read → decision → write path must be protected by conditional writes, transactions, unique constraints or deterministic state claims as appropriate.

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
