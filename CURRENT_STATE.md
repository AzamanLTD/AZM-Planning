# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact-head CI; this document is the navigation ledger.

> This file is intentionally short enough for an agent to read every session. It replaces the need to reconstruct state from dated handoffs.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main at `68bd1d51a28c61f31e556f616c488b2d7ce631a2` after PR #130 | PR #131 payroll retry; Business OS boundary/state audit; stale/duplicate PRs/branches |
| `AZM-adminPortal` | Main at `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state semantics; control-plane visibility |
| `AZM-businessPortal` | Main at `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority audit; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes PR #77 notification banner hardening | Full journey authority/realtime audit; dine-in contract first |
| `AZM-Planning` | This branch is reorganizing the planning brain | Merge canonical docs, then make `main` the obvious continuation source |

**Important:** verify these SHAs against GitHub before relying on them if another session has changed the repos.

## 2. Known verified recent backend hardening

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions made race-safe in PR #127.
- Time-off approval/rejection made tenant-scoped and concurrency-safe in PR #126.
- Payroll disbursement bounded P2034 retry is the purpose of PR #131; exact-head CI must be rechecked before merge.
- Generic shift PATCH still requires review to prevent direct `status` mutation.
- Shift rotation creation permission parity remains to be audited.
- Legacy `EmployeeService.updateAccruedWages()` requires consumer tracing before removal/restriction.

## 3. Known critical Admin state

PR #95 hardened Control Plane refresh interaction and exact-head CI passed. The Admin Portal remains the weakest tested high-risk surface: it owns or exposes force-release/cancel escrow, withdrawal approval/rejection, fee controls and War Room actions. New tests must be compatible with the repo's actual TypeScript/test configuration; never weaken type checking.

## 4. Known Business Portal state

PR #43 made invoice dashboard statistics authoritative. Earlier work also established authoritative revenue series and business-type routing. Continue auditing dashboard metrics for client-side aggregation, fixed limits, static fallbacks and duplicated calculations. `FinanceV2.jsx` and the large `marketplaceApi.js` require reachability/authority tracing before removal or refactoring.

## 5. Known Flutter state

Recent work hardened hotel inventory, transit fares, restaurant naming/selection, dine-in context/order and actionable notifications. Continue from existing abstractions. Do not duplicate backend payment/escrow authority in the app.

## 6. First implementation queue

1. **Backend PR #131:** verify exact-head CI; audit diff; merge if green and correct.
2. **Backend shift boundary:** remove generic `status` from `updateShift()` and reject direct status updates; add regression coverage.
3. **Backend route permissions:** audit/fix `POST /shifts/rotation` and attendance/EWA route-vs-service authorization semantics.
4. **Legacy wage mutation:** trace every consumer of `updateAccruedWages()` and prove whether it is redundant or still required before changing it.
5. **Dine-in contract:** trace Flutter/backend/business/admin producer-consumer contract and the finalize/payment race; implement concrete fixes, not a report.
6. **Admin critical tests:** lock withdrawal, escrow dispute, fee engine, War Room and dashboard contracts.
7. **Tenant/state sweep:** finish remaining Business OS and commerce object matrices.
8. **Realtime reconciliation:** define and implement missed-event convergence where absent.

## 7. Open risks that must remain visible

- Backend has substantial stale/duplicate branch/PR debt; exact counts from Lyra's assessment are a starting point and must be re-verified before deletion.
- Production deployment/staging path is not yet proven by the current planning evidence.
- Rollback of application + Prisma migration has not been demonstrated as a release rehearsal.
- Secret lifecycle/rotation is not yet proven.
- Realtime missed-event recovery is not yet a fully documented/verified mechanism.
- Dine-in is a high-value cross-repo money contract and should be audited before broad vertical work.
- Admin mutation test coverage remains a release blocker.
- Data migration safety for populated production tables is not yet a demonstrated discipline.
- Load testing has not yet been evidenced at the target minimums.

## 8. Session update protocol

At the end of each substantial batch, append/replace only the relevant state here with:

- what changed;
- repository + PR/commit;
- exact CI/test evidence;
- residual risk;
- next exact action.

Do not turn this into a long diary. Historical reasoning belongs in `archive/` or Git history.
