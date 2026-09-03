# AZAMAN Engineering Brain — START HERE

**Status:** ACTIVE — THIS IS THE PRIMARY CONTINUATION ENTRYPOINT  
**Last verified:** 2026-09-03 UTC  
**Repositories:** `AZM-backend`, `AZM-adminPortal`, `AZM-businessPortal`, `AZM-frontend`, `AZM-Planning`

> **Any agent continuing AZAMAN engineering MUST start here, then read `ROADMAP.md` and `CURRENT_STATE.md`. Do not use dated session journals as the active plan.**

## Mission

Bring AZAMAN to a production-ready state by removing ambiguity about **who owns truth, who may mutate it, how state transitions, how concurrency behaves, how failures recover, and how every client converges on authoritative outcomes**.

## Read order

1. **`ROADMAP.md`** — the total execution plan. This is the priority authority.
2. **`CURRENT_STATE.md`** — verified repository state, open work, blockers and the exact next batch.
3. **`CONTRACTS.md`** — cross-repo producer/consumer contracts and authority ownership.
4. **`FINANCIAL_INVARIANTS.md`** — money safety rules.
5. **`SECURITY_BOUNDARIES.md`** — authorization, tenant and actor boundaries.
6. **`STATE_MACHINES.md`** — lifecycle rules and legal/illegal transitions.
7. **`ARCHITECTURE.md`** — system map and major shared abstractions.
8. **`RELEASE_CHECKLIST.md`** — final production gate.

## Current execution priority

### P0 — do first

- Reconcile backend open PRs/duplicate branches before creating overlapping work.
- Finish payroll retry PR #131 verification/merge if still open.
- Harden Business OS shift mutation boundaries: generic PATCH must not directly mutate `status`; route permission gaps must be audited.
- Complete the remaining financial/concurrency audit, with **dine-in `confirmAndPay` as the first cross-repo contract audit**.
- Treat Admin Portal financial mutation coverage as a production blocker.
- Establish production deployment, rollback, migration and secret-lifecycle evidence.

### P1 — immediately after P0 foundations

- Tenant-boundary matrix across remaining Business OS and commerce surfaces.
- Full state-machine audit and regression tests.
- Cross-repo API/realtime contract matrix.
- Experience Blueprint contract: Business Portal configures → Backend owns → Flutter renders.
- Reconciliation/event recovery for missed realtime signals.

### P2/P3

- Marketplace runtime/dead-code cleanup.
- Business Portal component extraction and broader tests.
- Performance and UX/accessibility work, measured rather than speculative.

## Non-negotiable execution loop

**Research → trace producers/consumers → inspect schema/authorization/state machine → implement one coherent batch → test → exact-head CI → audit diff → merge → verify `main` → update Planning → continue.**

Never claim completion from code existence or a green CI run alone.

## Planning rules

- Repository `main` plus exact CI is implementation evidence; planning documents are navigation and intent.
- Never create duplicate implementation branches for the same logical fix.
- Before a new branch, inspect existing open PRs/branches and current `main`.
- Close/supersede stale PRs deliberately; delete stale branches after their work is resolved.
- Never weaken type checking or tests merely to make CI green.
- Preserve historical reasoning in `archive/` or Git history; do not allow historical documents to compete with the active plan.
- A session is not complete until `CURRENT_STATE.md` records what changed, evidence, remaining risks and the next exact action.

## If you only have one engineering turn

Read `ROADMAP.md` → read `CURRENT_STATE.md` → take the **first unchecked P0 item** → inspect actual code in all affected repos → implement and verify it → update `CURRENT_STATE.md`.
