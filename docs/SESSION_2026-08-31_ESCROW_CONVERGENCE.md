# 2026-08-31 — Escrow Concurrency Implementation Record

## Scope

This session completed the research pass required for SmartEscrow satisfaction concurrency and implemented the smallest canonical convergence fix in Backend PR #52.

## Research completed before implementation

Affected Backend surfaces:

- `services/escrowService.js`
- `controllers/escrowController.js`
- `__tests__/escrow-flow.test.js`
- new `__tests__/escrow-convergence.test.js`
- SmartEscrow database terminal-state guard/migration

Cross-portal consumers and convergence boundaries reviewed:

- Flutter `lib/services/escrow_service.dart`
- Flutter `lib/services/socket_service.dart`
- Flutter `lib/screens/tickets/ticket_workspace_screen.dart`
- Business Portal `src/lib/realtimeQueryBridge.js`
- Backend business notification/event paths
- existing `escrow_settled` producer/consumer paths

The financial invariant was confirmed before implementation: participant satisfaction uses conditional claims, `_releaseEscrow` has a single-winner conditional final-state claim, and the database blocks stale terminal-state regression.

## Root cause

The remaining defect was response convergence, not double-settlement protection.

A concurrent opposite-party request could reach a stale `PENDING_SETTLEMENT` write after another request had already committed `SETTLED`. The database correctly rejected the stale write, but the caller could receive a generic failure even though the escrow was already successfully settled.

A retry arriving after settlement had a similar problem because the original service rejected `SETTLED` before it could return the canonical state.

## Canonical implementation

PR #52 changes only three production/test files:

1. `services/escrowService.js`
   - `SETTLED` retries return `{ settled: true, alreadySettled: true }`.
   - satisfaction-claim losers re-read and converge if the escrow is already settled.
   - release-claim losers re-read and converge if another transaction committed `SETTLED`.
   - stale `PENDING_SETTLEMENT` writes re-read after a database rejection and converge if settlement committed concurrently.
   - no second settlement path is introduced.
   - the existing database terminal-state guard remains untouched.

2. `controllers/escrowController.js`
   - `alreadySettled` responses return HTTP 200 with the canonical escrow.
   - convergence responses do not emit a second `escrow_settled` socket event.
   - convergence responses do not create a duplicate settlement system TicketMessage.
   - a dedicated audit action records convergence.

3. `__tests__/escrow-convergence.test.js`
   - runs opposite-party satisfaction concurrently against a real disposable Postgres database;
   - requires both calls to resolve successfully;
   - verifies final state is `SETTLED` and both satisfaction flags are true;
   - verifies the payee receives exactly the escrow principal once;
   - verifies exactly one `TICKET_ESCROW_RELEASE` history row exists.

## Verification

A guarded GitHub Actions implementation workflow was used because the repository integration exposes whole-file writes but not an authenticated arbitrary patch operation.

The workflow:

- checked exact source anchors before modifying anything;
- installed the repository dependencies;
- applied the current Prisma schema to disposable Postgres;
- generated Prisma client;
- applied the guarded source/test patch;
- ran the existing escrow flow suite plus the new convergence regression;
- ran duplicate-settlement checks and `git diff --check`;
- removed the temporary workflow before pushing the implementation branch.

All implementation/test/duplicate-check steps completed successfully. The workflow's final `gh pr create` step failed because its token could not create a pull request. The branch was preserved and PR #52 was opened through the repository integration.

## Current PR

Backend PR #52:

`fix: converge concurrent escrow satisfaction responses`

Head: `6ff301600415f1b2d02b7c589269149c2e311949`

Base: Backend `main` at `ca2f16b38f8dbab913988b3eed767b1711b57448`

Changed files: 3

Production additions: 50
Production deletions: 7
Test additions: 60

The PR must pass the repository's normal full CI before merge. The focused implementation workflow is not a substitute for the normal mainline quality gate.

## Related newly verified issue

Backend issue #49 was expanded after a fresh producer/consumer audit.

Current main has:

- `businessOrderService.markDelivered`: `PAID -> DELIVERED`;
- `businessOrderController.markDelivered`: emits `order.completed` immediately after that delivery transition;
- `businessOrderService.updateOrderStatusFromEscrow`: `SETTLED/RELEASED -> COMPLETED`, including `DELIVERED -> COMPLETED`;
- `webhookEmitter.js` documentation lists `order.delivered` and `order.cancelled` but not `order.completed`;
- Business Portal already has singleton realtime/query invalidation behavior.

This is tracked as issue #49. No event rename/addition has been implemented yet. Before touching it, every producer and consumer across Backend, Business Portal, Admin Portal, and Flutter must be mapped.

## Next high-risk boundary

After PR #52 passes and is merged, re-research current main before starting the next task.

Priority order:

1. Admin force-release atomic financial boundary — Backend #48.
2. Order `delivered` vs `completed` webhook contract — Backend #49.
3. Ticket-scoped realtime subscription registry in Flutter without introducing a second socket.

For #48 specifically, do not use an unconditional `PAID -> DISPUTED` rollback. The correct fix must unify the force-release financial transaction boundary or use a conditional mechanism that cannot overwrite a newer state.

## Continuity rule

This record is evidence of what was actually researched and implemented in this session. It must be read alongside `MASTER_ARCHITECTURE_AND_EXECUTION_STATE.md` and `REALTIME_CONTRACT.md` before the next implementation.
