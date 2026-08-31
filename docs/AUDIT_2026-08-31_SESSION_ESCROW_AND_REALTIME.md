# AZM Engineering Audit — 2026-08-31

## Purpose

This audit records a research pass across Backend, Business Portal, Flutter, and Planning before the next implementation. It is intentionally factual: findings are tied to current `main` code and are not treated as implemented until merged and re-verified.

## Current baseline

- Backend main: `ca2f16b38f8dbab913988b3eed767b1711b57448`.
- Flutter escrow-entry PR #30 is merged at `3c2a2f9e6f8a700ec211d636e0048fd1005a74db`.
- Cross-repository open PR search currently returns no open PRs.
- Experimental Backend PR #50 was created only to test a safe repository-write workaround, did not execute its temporary workflow, was closed, and its branch was reset exactly to Backend main. It introduced no production change.

## Finding A — escrow concurrent response convergence

### Backend files researched

- `services/escrowService.js`
- `controllers/escrowController.js`
- `__tests__/escrow-flow.test.js`
- SmartEscrow database trigger/migration recorded in the master state

### Existing safety mechanisms

`markSatisfied` conditionally claims each participant's satisfaction flag using `updateMany`. `_releaseEscrow` separately performs a single-winner conditional status claim before moving any balance. The database also prevents a stale `PENDING_SETTLEMENT` write after a terminal settlement.

Therefore the financial invariant is already protected against double payout. Do not weaken or duplicate those protections.

### Remaining race

Two opposite-party satisfaction requests can interleave in three ways:

1. One request marks satisfaction and returns `PENDING_SETTLEMENT`; the other later settles normally.
2. Both observe both satisfaction flags and race `_releaseEscrow`; one wins and the other receives `ESCROW_ALREADY_FINALIZED` even though the escrow is already `SETTLED`.
3. A request attempts its stale `PENDING_SETTLEMENT` write after another request has committed `SETTLED`; the database guard correctly rejects the stale write, but the caller currently receives a generic failure.

### Canonical implementation decision

Preserve the existing financial claims and make the losing/retried request converge:

- If `markSatisfied` is retried after `SETTLED`, return `{ settled: true, alreadySettled: true, escrow: current }`.
- If `_releaseEscrow` loses its claim with `ESCROW_ALREADY_FINALIZED`, re-read the escrow; if it is `SETTLED`, return the committed state as convergence.
- Replace the unconditional `PENDING_SETTLEMENT` update with a conditional transition. If no row is claimed, re-read; if terminal `SETTLED`, return convergence; otherwise surface the real state conflict.
- Controller must not emit `escrow_settled` or inject a second system message when `alreadySettled` is true.
- Add a database-backed opposite-party concurrent test asserting both HTTP/service requests converge successfully and exactly one `TICKET_ESCROW_RELEASE` history row and one payee credit occur.

Prisma's transaction/OCC documentation supports this pattern: conditional updates are appropriate for concurrency control, and retries should converge to the canonical committed state rather than duplicate mutations.

## Finding B — Flutter escrow realtime listener architecture

### Files researched

- `lib/services/socket_service.dart`
- `lib/screens/tickets/ticket_workspace_screen.dart`

The singleton SocketService has one `_onEscrowEvent` callback. TicketWorkspace deliberately attaches ticket-specific handlers directly to the singleton's underlying socket because multiple ticket workspaces may coexist. It correctly removes only its own handler with `off(event, handler)`.

Replacing the screen handlers with `SocketService.onEscrowEvent()` would create last-subscriber-wins behavior because the singleton callback is a single slot. Do not make that change.

### Canonical future implementation

Add a subscription registry to SocketService:

- `subscribeEscrow(handler) -> unsubscribe handle`;
- filter by event and ticketId;
- preserve one underlying Socket.IO connection;
- replay subscriptions after reconnect without duplicate registration;
- exact unsubscribe during widget disposal;
- keep the global dispatcher separate from ticket subscriptions.

Then migrate TicketWorkspace from raw socket access to the registry and add tests for two simultaneous ticket subscribers, unsubscribe isolation, reconnect restoration, and duplicate event delivery.

## Finding C — Business Portal realtime boundary

### Files researched

- `src/lib/socket.js`
- `src/lib/realtimeQueryBridge.js`
- `services/bizNotificationService.js`

Business Portal already has one socket singleton and a singleton query bridge. The bridge removes exact handler references and rebinds on socket replacement, which is the correct pattern.

Escrow direct events invalidate orders broadly because their payloads carry `escrowId`/`ticketId` rather than `orderId`. The durable `biz_notification` signal carries `orderId`, `ticketId`, and `escrowId`, so the notification path provides precise order convergence after the persisted BusinessNotification is created.

Do not introduce a second socket or another notification event family. If order precision is improved, extend the existing bridge/event contract rather than adding parallel events.

## Finding D — Admin force release

### Files researched

- `controllers/adminController.js`
- `services/p2p.service.js`

Admin force release conditionally changes `DISPUTED → PAID` and then calls `completeTrade`. `completeTrade` itself atomically claims `PAID → COMPLETED` before balance/ledger writes. The controller comment already documents the stranded-`PAID` failure mode.

The correct future implementation is not an unconditional catch rollback. It must unify the financial transaction boundary or use a conditional compensation that cannot overwrite a newer committed state. All completeTrade callers, audit writes, events, and Admin Portal response handling must be researched again immediately before implementation.

## Finding E — order webhook semantics

The business order lifecycle distinguishes `PAID → DELIVERED` from later completion. The external webhook contract must not emit `order.completed` for the delivery transition. Before changing anything, audit every producer and consumer across Backend, Business Portal, Admin Portal, Flutter, documentation, and tests.

Never solve the mismatch by emitting both event names for one state transition.

## Repository-write workaround result

The GitHub connector can safely create branches/files and construct commits, but its file-update operation requires complete replacement content. A temporary self-removing GitHub Actions workflow was tested as a guarded patch mechanism; no workflow run was available for the repository, so the experiment was stopped. The temporary PR was closed and its branch was reset to the exact main commit. No production code was changed by that experiment.

Future code changes must use a verified repository write path that can apply guarded patches and run the repository's real CI. Do not leave temporary automation, partial source replacements, or experimental branches open.

## Next implementation order

1. Escrow concurrent-response convergence.
2. Exact Backend escrow regression and full test gate.
3. Re-audit all escrow event producers/consumers and duplicate emissions.
4. Flutter multi-subscriber escrow registry.
5. Admin force-release atomic transaction boundary.
6. Order webhook semantic correction.
7. Continue through the master roadmap with the same research → implementation → verification → post-merge audit loop.
