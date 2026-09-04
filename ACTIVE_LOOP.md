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

### Verified on current GitHub main

- Backend PR #131 payroll Serializable retry merged/verified.
- Backend PR #132 shift generic-status mutation boundary merged/verified.
- Backend PR #135 dine-in settlement/finalization/payment-idempotency hardening merged.
- Backend PR #136 KYB gate fail-closed hardening merged.
- Backend PR #137 canonical Business OS Finance routing/runtime repair merged/verified.
- Backend PR #141 inventory restock atomicity merged/verified.
- Backend PR #143 corrected the real dine-in test mock failure.
- Backend PR #144 kiosk scoped capability + PIN rate limiting merged.
- Backend PR #145 POS settlement/inventory atomicity merged.
- Backend PR #146 duplicate-line recipe consumption merged.
- Backend PR #147 transaction-time POS catalog authority merged; exact-head Actions run `33778925260` succeeded through tests and database recovery drill.
- Backend PR #148 POS location/table/product boundary hardening merged and verified.
- Backend PR #150 POS idempotency request-intent binding merged as `7bc3f142b7db074843016cd01d21eae070041717`.
- Backend PR #151 dine-in location/table/branch-product boundary hardening merged as `f2002fd5681aaa5572c84672ded3adc23c511d72`; exact-head Actions run `33819904137` succeeded.
- Backend PR #152 legacy/locationless dine-in product boundary merged as `e2f9a8e29cf760a2b43e0f2f3429a87ad296391a`; exact-head Actions run `33821697996` succeeded through tests and database recovery drill.
- Frontend PR #78 dine-in payment failure truthfulness merged/verified.
- Business Portal PR #44 customer payment authority fix merged.
- Stale Planning PR #27 closed after becoming obsolete.

### Backend main

`e2f9a8e29cf760a2b43e0f2f3429a87ad296391a`

### Active implementation

#### Tax preset tenant boundary — PR #155

- Branch: `fix/tax-preset-tenant-boundary-v2`.
- Head: `ca553255a4ae177f46f8fdd8cace878ce4c542aa`.
- The BusinessTaxPreset PATCH handler historically updated by bare preset ID; the new resource-level guard in `requirePermission` verifies `{id, businessProfileId}` before the handler runs.
- Exact-head full CI + database recovery drill required before merge.
- Stale PR #153 was intentionally closed after PR #152 advanced main; the same fix was recreated cleanly on current main.

#### POS global catalog boundary — PR #154

- Branch: `fix/pos-global-product-boundary`.
- Head: `2a97cdb52f27f1d2bd003cb1fba709364141e71f`.
- Locationless POS requests now resolve only globally scoped products. Explicit location requests still allow global or exact-location products.
- Exact-head full CI + database recovery drill required before merge.

### Next P0 after #154/#155

1. Resolve POS/invoice tax authority by completing the producer/consumer trace for `BusinessInvoice`, `BusinessInvoiceTaxLine`, and `BusinessTaxPreset`; only then change legacy tax behavior.
2. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, especially replay-after-payment closure, tips, timeout recovery, and location/table context.
3. Bring Business Portal dine-in UI/API onto the location-aware server contract; its current client path still opens tabs without location context and exposes a legacy tax-rate control that the adapter drops.
4. Trace every `updateAccruedWages()` producer/consumer/history before removal or restriction.
5. Strengthen Admin financial mutation and optimistic pending-queue coverage, then run tenant/state/realtime, production operations, load and red-team waves.

## CI / release gate

Backend Actions executes normally on the public repositories. The required gate remains: exact PR head, full test stage, and database backup/restore drill must be green before merge.

## Duplicate-work prohibition

Do not resurrect superseded POS #138/#140 or kiosk #142 work, and do not reopen Planning #27. Build from current `main` and consolidate into the canonical active path.

## Continuation rule

When CI is running, continue independent audits instead of waiting idle. When CI completes, return to the exact-head gate. After every verified merge, reconcile `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`, then continue to the next P0.
