# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file before engineering. This file is deliberately short; historical reasoning is preserved in Git history/archive rather than mixed into active instructions.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `68bd1d51a28c61f31e556f616c488b2d7ce631a2` after PR #130 | PR #131; Business OS boundaries/state; stale/duplicate PRs/branches |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes PR #77 notification banner hardening | Full journey authority/realtime audit; dine-in contract first |
| `AZM-Planning` | Main `da489d71fbe76f411494315c7e20f2c85d1bbc4b` includes canonical planning-brain consolidation | Keep canonical docs current; no new session-journal clutter |

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

## 3. Known verified backend hardening

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions made race-safe in PR #127.
- Time-off approval/rejection made tenant-scoped and concurrency-safe in PR #126.
- Payroll disbursement bounded P2034 retry is PR #131 and requires exact-head CI verification before merge.
- Generic shift PATCH status boundary remains an immediate fix.
- Backend PR #133 now covers shift rotation permission parity, manager attendance route permissions and Business OS EWA management route permissions; exact-head CI and merge verification remain required.
- Legacy `EmployeeService.updateAccruedWages()` requires consumer tracing before removal/restriction.

## 4. Weakest/highest-risk surfaces

**Admin Portal:** small test surface relative to force-release/cancel escrow, withdrawal approval/rejection, fee controls and War Room actions. This is a production blocker, not optional cleanup.

**Dine-in:** first cross-repo contract audit because it crosses Flutter, Backend and Business Portal, moves real money and has a finalize/payment race.

**Production operations:** staging/deployment, rollback, secret rotation, populated-data migration safety, monitoring and load evidence are not yet proven.

**Engineering hygiene:** stale/duplicate branches and open PRs must be reconciled before overlapping implementation is started.

## 5. First implementation queue

1. Verify exact-head CI for Backend PR #131; audit and merge if correct.
2. Harden `ShiftService.updateShift()` so generic PATCH cannot mutate `status`; add regression test.
3. Audit/fix `POST /shifts/rotation` permission parity and document attendance/EWA route-vs-service authorization.
4. Trace all `updateAccruedWages()` consumers and remove/restrict only after proving redundancy/safety.
5. Deep-audit the dine-in contract and finalize/payment concurrency; implement concrete fixes.
6. Add Admin financial mutation tests: withdrawals, escrow disputes, fee engine, War Room, dashboard.
7. Complete tenant/state matrices for remaining Business OS and commerce objects.
8. Implement/verify realtime missed-event reconciliation.

## 6. Lyra findings incorporated into the active plan

The canonical roadmap explicitly incorporates: branch/PR debt; deployment and rollback; secret lifecycle; production migration discipline; multi-surface partial-failure recovery; dine-in priority; Admin test debt; Experience Blueprint contract; and minimum load/red-team expectations.

## 7. Open risks

- Exact stale-branch counts must be re-verified before bulk deletion.
- Deployment/staging process is not yet proven by current evidence.
- Code + Prisma migration rollback/recovery has not been rehearsed.
- Secret lifecycle/rotation is not yet proven.
- Realtime missed-event recovery is not yet fully demonstrated.
- Admin critical mutation coverage remains thin.
- Populated-production migration safety is not yet demonstrated.
- Load-testing evidence is absent at the target minimums.

## 8. Session update protocol

Every substantial engineering batch must update this file with: change, repo, PR/commit, exact tests/CI evidence, residual risk and next exact action. Never mark work VERIFIED from discussion alone.
