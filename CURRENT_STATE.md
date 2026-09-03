# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file → `ACTIVE_LOOP.md` → `EXECUTION_LEDGER.json` before engineering. Historical reasoning belongs in Git history/archive, not here.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `924807b3742f30f929479d46bda96d9660b61f2d` after PR #137 | POS/inventory/kiosk changes remain unmerged; fresh Actions runs are intermittently failing before executable steps are exposed |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes verified PR #78 failure propagation | Continue dine-in client/server convergence audit; verify end-to-end payment/retry semantics |
| `AZM-Planning` | This reconciliation is being merged through a dedicated branch/PR | Keep continuation state synchronized after every substantial verified batch |

**Never rely on these SHAs after another session without reconciling them against GitHub.**

## 2. Planning-brain consolidation — VERIFIED

PR #24 (`docs: establish canonical engineering brain`) merged at `da489d71fbe76f411494315c7e20f2c85d1bbc4b`. Persistent continuation files are present on `main`: `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`. `ROADMAP.md` remains the single active execution authority.

## 3. Verified backend hardening and current batch

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions race-safe in PR #127.
- Time-off approval/rejection tenant-scoped and concurrency-safe in PR #126.
- Payroll bounded P2034 retry merged/verified in PR #131.
- Shift generic PATCH status boundary merged/verified in PR #132.
- Dine-in settlement/finalization/idempotency hardening merged in PR #135 at `fc24fcc61800e86cad9657b91b59c8d24e93a8ef`.
- KYB gate fail-closed hardening merged in PR #136.
- Business OS finance canonical routing/runtime repair merged in PR #137 at main `924807b3742f30f929479d46dba96d9660b61f2d`.

### POS — current PR #140

- PR #138 is closed/superseded and was never merged.
- PR #140 is open on `fix/business-os-pos-atomicity-v2` at head `6a2947447aec60528033d1ef0a7416bebf5b2b05`; GitHub currently reports `mergeable=false` and `merged=false`.
- Implementation server-derives catalog pricing, validates integer quantities, supports CASH/AZM/SPLIT, conditionally debits AZM, persists `BusinessOrderItem` rows, and commits order/line-items/ledger atomically with Serializable retry. Idempotent replay resolves before catalog validation while enforcing tenant ownership.
- Legacy tracing confirms the existing `/pos/order` used the legacy 2.5% POS tax and `customerId || authenticatedUserId` fallback; these are compatibility findings, not proof that those semantics are the final desired accounting contract.
- **Known remaining risk:** 2.5% tax is hardcoded even though the backend has `BusinessTaxPreset`; inventory decrement, location/table integrity, customer identity semantics and payload-binding for idempotency still need authoritative contract evidence.
- No exact-head green workflow run is currently attached to `6a294744...`; do not merge without a full exact-head `Azaman Test Suite` success.

### Inventory — current PR #141

- Older PR #139 is closed/superseded and was never merged.
- PR #141 is open on `fix/business-os-inventory-restock-atomicity-v2` at current head `fc5b1df14a99a4895860649b850f7c575265b42e`; GitHub reports `mergeable=true`, `merged=false`.
- The PR is intentionally independent of the unmerged POS router. It adds a canonical scoped restock route and service with positive-quantity/non-negative-cost validation and a single transaction for stock increment plus signed `SUPPLIES` expense; ledger failure propagates and therefore rolls the transaction back.
- Earlier runs reached the real test stage and failed; the newest run `33749730364` (head `fc5b1df...`) completed in roughly three seconds with no executable steps exposed and no log blob, strongly indicating an Actions job-startup/runner failure rather than a reproduced Jest failure.
- Temporary diagnostic workflows were created only to capture the unavailable Jest output and were removed again; they are not part of the PR payload.
- Do not merge until a current exact-head full test run executes and passes.

### Kiosk — current PR #142

- PR #142 is open on `fix/business-os-kiosk-capability` at head `40c78279c87e818071f0b9317149e96904ddc3eb`; GitHub reports `mergeable=true`, `merged=false`.
- Capability signing/verification is isolated; clock-in/out enforce tenant/employee/user/shift binding and location binding; PIN auth validates that a supplied location belongs to the business. Focused tests cover expiry, scope, tenant, employee and location binding.
- Exact-head run `33746899789` failed before executable steps were exposed. Do not merge without exact-head green evidence.

## 4. CI/release-gate evidence

The canonical Backend workflow remains unchanged. Known-good PR #137 run `33732480681` executed setup, tests and the DB recovery drill successfully. Recent active PR attempts have two distinct evidence classes: some older attempts ran the Jest stage and failed, while the newest active-branch attempts fail almost immediately with zero exposed steps and unavailable log blobs. This is an execution-reliability blocker, **not** permission to weaken the release gate.

Current exact references:

- Inventory latest observed head: `fc5b1df14a99a4895860649b850f7c575265b42e`
- Inventory latest run: `33749730364` — failure, no executable steps exposed
- POS current head: `6a2947447aec60528033d1ef0a7416bebf5b2b05` — no exact-head workflow evidence
- Kiosk current head: `40c78279c87e818071f0b9317149e96904ddc3eb` — latest observed run `33746899789`, zero-step failure

## 5. Frontend dine-in payment — VERIFIED

PR #78 was verified with Flutter Quality run `33725099978` / run #314 and merged as `7cf85fa1942c277f5b2de4578e41d75cf81b20a5`. `DineInTabNotifier.payTab()` now records failure state and rethrows, so the UI cannot falsely treat a failed payment as success.

This closes the frontend failure-truthfulness item only. The broader cross-repo `confirmAndPay` audit remains open for payment authority, idempotency, tab closure, tips, realtime and reconciliation.

## 6. Active P0 queue

1. Obtain reliable exact-head CI execution for POS #140, inventory #141 and kiosk #142 without weakening the release gate; merge only after green exact-head evidence.
2. Complete POS inventory/ledger/location/replay/tax semantics audit.
3. Complete kiosk abuse-resistance/rate-limit review.
4. Trace every `updateAccruedWages()` consumer/history before removal or restriction.
5. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter.
6. Add Admin financial mutation coverage and correct optimistic pending-queue behavior.
7. Complete tenant/state/realtime matrices, then production-ops/load/red-team waves.

## 7. Open risks

- Exact-head CI reliability is unresolved for active backend PRs.
- POS hardcoded 2.5% tax may not be the final accounting authority despite matching legacy behavior; `BusinessTaxPreset` exists and must be reconciled before production confidence.
- POS inventory decrement and location/table integrity remain open.
- Inventory restock currently has no structural idempotency key; do not add one until retry/replay semantics and schema authority are established.
- Kiosk PIN enumeration/rate limiting remains a hardening item.
- Dine-in end-to-end reconciliation and realtime remain open despite frontend error propagation fix.
- Deployment/staging, migration rollback, secret lifecycle, realtime recovery, Admin mutation coverage and load-testing evidence remain open.

## 8. Session update protocol

Every substantial engineering batch must update this file plus `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` with exact repo/PR/head, CI evidence, residual risk and next action. Never mark work VERIFIED from discussion alone.

**2026-09-03 Backend CI reconciliation:** PR #141 was corrected to remove its dependency on the unmerged POS router; the branch now contains only its intended four files. Latest exact-head run `33749730364` failed at job startup with no exposed steps, while earlier attempts reached the test stage and failed. PR #140 remains open and unverified; PR #142 remains open and unverified. The release gate is unchanged.
