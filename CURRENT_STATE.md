# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file before engineering. This file is deliberately short; historical reasoning is preserved in Git history/archive rather than mixed into active instructions.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `924807b3742f30f929479d46bda96d9660b61f2d` after PR #137 | POS/inventory/kiosk changes remain unmerged; exact-head CI execution is unreliable |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main now includes verified PR #78 failure propagation in addition to PR #77 | Continue dine-in client/server convergence audit; verify end-to-end payment/retry semantics |
| `AZM-Planning` | Main contains reconciliations through PR #30; this frontend-payment reconciliation is on a dedicated branch until merged | Keep continuation state synchronized after every substantial batch |

**Verify current application SHAs against GitHub before relying on them if another session has changed the repos.**

## 2. Planning-brain consolidation — VERIFIED

PR #24 (`docs: establish canonical engineering brain`) was merged at `da489d71fbe76f411494315c7e20f2c85d1bbc4b` after an exact-head merge. `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` provide persistent continuation state; `ROADMAP.md` remains the single active execution authority.

## 3. Verified backend hardening and current batch

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions made race-safe in PR #127.
- Time-off approval/rejection made tenant-scoped and concurrency-safe in PR #126.
- Payroll disbursement bounded P2034 retry merged/verified in PR #131.
- Shift generic PATCH status boundary merged/verified in PR #132.
- Business OS finance canonical routing/runtime repair merged in PR #137 at main `924807b3742f30f929479d46bda96d9660b61f2d`.
- Dine-in settlement/finalization/idempotency hardening merged in PR #135 at `fc24fcc61800e86cad9657b91b59c8d24e93a8ef`; cross-repo reconciliation remains active.
- KYB gate fail-closed hardening merged in PR #136.

### POS — current replacement PR #140

- PR #138 is closed/superseded and was never merged.
- PR #140 is open on `fix/business-os-pos-atomicity-v2` at head `6a2947447aec60528033d1ef0a7416bebf5b2b05`.
- Server-authoritative catalog pricing, integer quantity validation, CASH/AZM/SPLIT, conditional AZM debit, authoritative `BusinessOrderItem` persistence, order/line-items/ledger transactionality and replay-safe idempotency ordering are implemented.
- Legacy producer tracing confirms the existing `/pos/order` uses the same 2.5% POS tax behavior and `customerId || authenticatedUserId` fallback. A focused cash/tax regression was added on the current head.
- Exact-head CI remains unproven: the current head has no attached pull-request workflow run. Do not merge until a full exact-head `Azaman Test Suite` succeeds.

### Inventory — current replacement PR #141

- Older duplicate PR #139 is closed and not merged; PR #141 is the sole current inventory path.
- PR #141 is open on `fix/business-os-inventory-restock-atomicity-v2` at head `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`.
- Stock mutation and signed SUPPLIES expense are transactional with business/quantity/cost validation and ledger-failure coverage.
- Exact-head runs `33746032891`, `33746106832`, `33746223780`, `33746364784` (rerun attempt 2) failed before executable job steps were exposed. Do not merge on that evidence.

### Kiosk — current PR #142

- PR #142 is open on `fix/business-os-kiosk-capability` at head `40c78279c87e818071f0b9317149e96904ddc3eb`.
- Capability signing/verification is isolated; clock-in/out enforce tenant/employee/user/shift binding and optional location binding; PIN auth validates a supplied business location.
- Focused tests cover expiry, scope, tenant, employee and location binding.
- Exact-head CI run `33746899789` (#608), including rerun attempt, fails before executable steps are exposed. Do not merge without exact-head green evidence.

### CI/release-gate blocker

The canonical Backend workflow remains unchanged. Known-good PR #137 run `33732480681` executed setup, tests and DB recovery successfully. Fresh active PR heads can fail at job startup/no-step execution, while earlier attempts also reached the real test stage and failed. The release gate must remain unchanged; diagnose execution reliability rather than weakening tests.

## 4. Frontend dine-in payment — VERIFIED

PR #78 (`fix(dine-in): propagate payment failures`) was verified against exact head `ce5f29b4d6f52053088693df8c81aa544f5a1ad8`. Flutter Quality run `33725099978` / run #314 passed setup, dependency install, analyze, tests with coverage and upload. The diff adds the required `rethrow` after setting error state in `DineInTabNotifier.payTab()`. PR #78 was merged as `7cf85fa1942c277f5b2de4578e41d75cf81b20a5`, and `main` was re-read to confirm the `rethrow` is present.

This closes the previously unverified frontend failure-truthfulness claim. It does **not** close the broader cross-repo dine-in audit: Backend `confirmAndPay` plus Business Portal/Flutter response, idempotency, tab closure, tips, realtime and retry/reconciliation still need evidence.

## 5. First implementation queue

1. Resolve reliable exact-head CI execution for POS #140, inventory #141 and kiosk #142 without weakening the release gate; merge only after exact-head green evidence.
2. Finish POS inventory/ledger/location/replay semantics audit.
3. Finish kiosk abuse-resistance/rate-limit review and exact-head CI.
4. Trace every `updateAccruedWages()` consumer/history before removal or restriction.
5. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter, now with client failure propagation verified.
6. Add Admin financial mutation tests and correct optimistic pending-queue behavior.
7. Complete tenant/state/realtime matrices, then production-ops/load/red-team waves.

## 6. Open risks

- Exact-head CI remains unproven for POS #140, inventory #141 and kiosk #142 because fresh runs fail before executable steps or do not attach to the latest connector-written head.
- POS inventory-decrement semantics and location/table integrity still require producer/consumer evidence.
- Inventory restock has no structural idempotency key; schema/contract authority is required before adding one.
- Kiosk PIN enumeration/rate limiting should be strengthened beyond the existing general route limiter if exposed broadly.
- Dine-in backend/client end-to-end retry/reconciliation and tip authority remain open despite the frontend failure-path fix.
- Deployment/staging, migration rollback, secret lifecycle, realtime recovery, Admin mutation coverage and load-testing evidence remain open.

## 7. Session update protocol

Every substantial engineering batch must update this file with change, repo, PR/commit, exact tests/CI evidence, residual risk and next exact action. Never mark work VERIFIED from discussion alone.

**2026-09-03 frontend payment reconciliation:** Flutter PR #78 is now verified and merged; `main` contains the failure-propagating `payTab()` behavior. Planning retains the Backend exact-head CI blocker as an active release gate.
