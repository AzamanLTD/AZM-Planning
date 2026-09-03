# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file before engineering. This file is deliberately short; historical reasoning is preserved in Git history/archive rather than mixed into active instructions.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `924807b3742f30f929479d46dba96d9660b61f2d` after PR #137 | POS and inventory atomicity PRs remain gated; CI runner/job-startup failures need resolution; then continue Wave 1/2 audits |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes PR #77 notification banner hardening | Re-verify payment-failure semantic fix claim; full journey authority/realtime audit; dine-in contract first |
| `AZM-Planning` | Main remains `da489d71fbe76f411494315c7e20f2c85d1bbc4b` pending this reconciliation PR | Keep this file current via one Planning PR per substantial batch; no session-journal clutter |

**Verify current application SHAs against GitHub before relying on them if another session has changed the repos.**

## 2. Planning-brain consolidation — VERIFIED

PR #24 (`docs: establish canonical engineering brain`) was merged at `da489d71fbe76f411494315c7e20f2c85d1bbc4b` after an exact-head merge. The active planning surface is now:

- `START_HERE.md` — mandatory continuation entrypoint;
- `ROADMAP.md` — single active execution/priority authority;
- `CURRENT_STATE.md` — live state/evidence ledger;
- `ARCHITECTURE.md` — system/authority map;
- `CONTRACTS.md` — cross-repo contracts;
- `FINANCIAL_INVARIANTS.md` — money safety;
- `SECURITY_BOUNDARIES.md` — authorization/tenant/actor boundaries;
- `STATE_MACHINES.md` — lifecycle rules;
- `RELEASE_CHECKLIST.md` — production gate;
- `REPO_GUIDE.md` — continuation protocol;
- `archive/` — historical policy/index plus preserved Git history.

Superseded dated root planning files were removed from the active surface; their exact historical content remains recoverable from Git history. This is intentional: fewer competing instructions, not loss of provenance.

## 3. Verified backend hardening and current batch

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions made race-safe in PR #127.
- Time-off approval/rejection made tenant-scoped and concurrency-safe in PR #126.
- Payroll disbursement bounded P2034 retry is PR #131 and requires exact-head CI verification before merge.
- Shift generic PATCH status boundary is implemented in PR #132 and requires the established exact-head release gate where still pending.
- Business OS finance canonical routing/runtime repair merged in PR #137 at main `924807b3742f30f929479d46dba96d9660b61f2d`.
- Dine-in settlement/finalization/idempotency hardening merged in PR #135; follow-up cross-repo reconciliation remains active.
- KYB gate fail-closed hardening merged in PR #136.

### Current POS/inventory batch

- Former POS PR #138 is **closed and superseded**, not merged. It accumulated diagnostic/CI workflow churn and is retained only for audit history.
- POS replacement PR #140 (`fix(pos): atomic Business OS POS settlement (current main)`) is open on branch `fix/business-os-pos-atomicity-v2`. Current head is `c1e4186caac0161fea41f253fd96703bbb3980e8`.
- PR #140 now has canonical CI workflow only (obsolete diagnostic workflow removed), server-authoritative product pricing, integer quantity bounds, CASH/AZM/SPLIT handling, conditional AZM debit, idempotency protection, order+ledger atomicity, and authoritative `BusinessOrderItem` persistence inside the same transaction.
- PR #140 still has **no fresh exact-head green release run**. Its earlier PR check (#584) targeted an older commit and failed before normal job steps; subsequent integration-written commits did not produce a new pull-request run for the current head. Do not merge until an exact-head green run exists.
- Inventory restock PR #139 remains open at head `f3c945204333ccd6074fcfd6571e7336386f249e`. Its service now writes both signed ledger `amount` and GHS-equivalent `amountGhs` in the same transaction as stock mutation. The prior unit-test contract mismatch is fixed.
- PR #139 also lacks a fresh exact-head green release run; recent failures ended before executable job steps were recorded. Do not merge until exact-head CI is green.
- The known-good Backend release run `33732480681` on PR #137 executed all test-suite stages successfully, proving the canonical workflow can run; current zero-step failures are therefore not proof of code failure.

## 4. Weakest/highest-risk surfaces

**CI/release gating:** current PR integration is not producing reliable exact-head pull-request runs for newly written heads. This must be resolved without weakening the release gate or bypassing tests.

**Admin Portal:** small test surface relative to force-release/cancel escrow, withdrawal approval/rejection, fee controls and War Room actions. This is a production blocker, not optional cleanup.

**Dine-in:** first cross-repo contract audit because it crosses Flutter, Backend and Business Portal, moves real money and has a finalize/payment race.

**Production operations:** staging/deployment, rollback, secret rotation, populated-data migration safety, monitoring and load evidence are not yet proven.

**Engineering hygiene:** stale/duplicate branches and open PRs must be reconciled before overlapping implementation is started.

## 5. First implementation queue

1. Resolve the exact-head CI execution problem for Backend PR #140/#139 without weakening the release gate; obtain green evidence before any merge.
2. Audit POS contract against legacy producer/consumer semantics: tax authority, line-item semantics, inventory effects, payment/customer identity, ledger units, and idempotency replay behavior.
3. Verify/reconcile PR #139 against the eventual POS/main head, then exact-head CI and merge only when proven green.
4. Finish kiosk capability hardening and exact-head verification.
5. Trace all `updateAccruedWages()` consumers and remove/restrict only after proving redundancy/safety.
6. Deep-audit the dine-in contract and finalize/payment concurrency across Backend, Business Portal and Flutter.
7. Add Admin financial mutation tests: withdrawals, escrow disputes, fee engine, War Room, dashboard.
8. Complete tenant/state matrices for remaining Business OS and commerce objects.
9. Implement/verify realtime missed-event reconciliation.
10. Move into production-ops, load and red-team waves only after the investor-critical mutations have clean release evidence.

## 6. Lyra findings incorporated into the active plan

The canonical roadmap explicitly incorporates: branch/PR debt; deployment and rollback; secret lifecycle; production migration discipline; multi-surface partial-failure recovery; dine-in priority; Admin test debt; Experience Blueprint contract; and minimum load/red-team expectations.

No separate Lyra execution channel/tool is available to this agent. External/concurrent engineering activity must be treated as repository state and independently verified before acceptance.

## 7. Open risks

- Current exact-head CI cannot yet be proven for the newly written POS/inventory heads because GitHub pull-request workflow runs are stale/missing or fail before job steps begin.
- POS tax authority is not yet established; current implementation uses the existing 2.5% behavior but this remains a contract-audit item before release.
- POS inventory-decrement semantics and business-unit ledger conventions require final producer/consumer verification.
- Inventory restock has no structural idempotency key yet; adding one requires schema/contract authority before implementation.
- Deployment/staging process is not yet proven by current evidence.
- Code + Prisma migration rollback/recovery has not been rehearsed.
- Secret lifecycle/rotation is not yet proven.
- Realtime missed-event recovery is not yet fully demonstrated.
- Admin critical mutation coverage remains thin.
- Populated-production migration safety is not yet demonstrated.
- Load-testing evidence is absent at the target minimums.

## 8. Session update protocol

Every substantial engineering batch must update this file with: change, repo, PR/commit, exact tests/CI evidence, residual risk and next exact action. Never mark work VERIFIED from discussion alone.

**2026-09-03 batch note:** This reconciliation records the concrete POS/inventory fixes and the CI-gating anomaly discovered in the current session. It intentionally does not mark either PR merged or verified; the next agent must begin by reading `START_HERE.md`, `ROADMAP.md`, then this file and re-checking the live GitHub PR heads/runs.
