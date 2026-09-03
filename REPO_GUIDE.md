# AZAMAN Repository Guide

| Repo | Start with | High-risk areas |
|---|---|---|
| `AZM-backend` | routes → services → Prisma schema → tests | money, tenant scope, state transitions, events |
| `AZM-frontend` | route/providers/services → journey screens | payment authority, cached state, realtime, UX |
| `AZM-businessPortal` | pages → API modules → React Query | KPI authority, mutations, Blueprint, client calculations |
| `AZM-adminPortal` | mutation components → `financialApi` → hooks | escrow/withdrawal/fee/War Room actions |
| `AZM-Planning` | `START_HERE.md` → `ROADMAP.md` → `CURRENT_STATE.md` | stale instructions, duplicate plans |

## Agent continuation protocol

1. Read `START_HERE.md`, `ROADMAP.md`, `CURRENT_STATE.md`.
2. Inspect current `main` in every affected repo.
3. Search all open PRs and branches for the intended fix before creating a branch.
4. Trace producer → transport → consumer for cross-repo changes.
5. Trace authorization → tenant scope → state transition → side effects for mutations.
6. Inspect schema/constraints/transaction boundaries.
7. Implement one coherent batch on a newly created branch.
8. Run focused tests, then exact-head CI.
9. Audit the final diff and verify `main` after merge.
10. Update `CURRENT_STATE.md` and relevant living contract/invariant doc.
11. Delete/supersede the branch/PR when the work is resolved.
12. Continue with the first unchecked P0 item; do not stop at a report.

## Evidence standard

Use these labels consistently:

- **PLANNED:** intended, not implemented.
- **IN PROGRESS:** active implementation.
- **IMPLEMENTED:** code exists, not fully verified.
- **VERIFIED:** current mainline behavior plus appropriate tests/CI/evidence.
- **BLOCKED:** cannot safely proceed; record exact blocker.
- **DEFERRED:** intentionally postponed with reason.
- **REJECTED:** investigated and intentionally not pursued.

Discussion, an old handoff, or a passing test that does not exercise the relevant path is not VERIFIED evidence.
