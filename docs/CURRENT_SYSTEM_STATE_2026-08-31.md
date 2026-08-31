# AZAMAN Current System State — 2026-08-31

## Purpose

This is a point-in-time verification ledger for continuation sessions. It does not replace `MASTER_ROADMAP_2026-08-30.md`; it records what has been verified after the latest engineering loop and prevents stale handoffs from causing duplicate implementation.

## Verified implementation state

### Financial / trust

- Backend withdrawal lifecycle hardening is merged: canonical transaction identity, atomic worker claiming, PROCESSING reconciliation, network propagation, and provider-label compatibility.
- Backend invoice payment idempotency hardening is merged: atomic claim and committed replay behavior.
- Flutter invoice payment/receipt convergence and customer order-delivery convergence are merged.
- Business Portal invoice void convergence is merged.
- Socket events remain convergence signals; canonical HTTP/API projections remain authoritative.

### Retail escrow

The previous assessment's statement that there was no Flutter escrow funding UI is now stale.

Research of the current mainline found that:

1. Retail cart checkout creates a Ticket and a `SmartEscrow` in `DRAFT` atomically when `paymentMode=ESCROW`.
2. The order stores both `escrowId` and `ticketId`.
3. Flutter's `TicketWorkspaceScreen` loads `escrowProvider` for ESCROW tickets.
4. `EscrowStatusPanel` already provides the complete Fund Escrow interaction, including confirmation and biometric authentication.
5. `EscrowService.fundEscrow()` calls the canonical `/escrow/fund` endpoint.
6. Backend `/escrow/fund` is protected by active-user guard, 2FA, idempotency middleware and validation.

The actual missing product boundary was discoverability: `MyOrdersScreen` did not explicitly expose that an `AWAITING_PAYMENT` order with an escrow/ticket was awaiting funding. A small Flutter PR (#30) now adds a `Fund escrow` CTA that reuses the existing TicketWorkspace flow. It intentionally does not duplicate payment logic or create a second escrow service.

## Important newly discovered correctness risk

`escrowService.markSatisfied()` currently claims the participant satisfaction flag with a conditional update, then reads the row, and if both flags are true calls the single-winner release routine. If only one party is satisfied, it subsequently performs an unconditional `status = PENDING_SETTLEMENT` update.

Under a concurrent two-party satisfaction race, this ordering can permit:

- party A claims payerSatisfied;
- party B claims payeeSatisfied and settles the escrow;
- party A's earlier read still observes the other flag as false;
- party A then writes `PENDING_SETTLEMENT` after the settlement commit.

The single-winner release guard prevents double payout, but it does not by itself prevent this terminal-state regression. This must be fixed before declaring the escrow lifecycle complete.

The correct implementation should make the satisfaction transition and terminal settlement decision atomic, or use conditional status updates that cannot overwrite `SETTLED`/other terminal states. Do not add a second state machine or a second financial release implementation.

## Important order/webhook contract risk

The business order service uses `PAID → DELIVERED → COMPLETED`, while the current controller historically emitted `order.completed` immediately after delivery. The webhook emitter's documented canonical event is `order.delivered`.

Before changing this, trace every webhook producer/consumer and establish one authoritative event mapping:

- `PAID → DELIVERED` → `order.delivered`
- `DELIVERED → COMPLETED` → `order.completed`

Do not emit both names for the same transition merely to improve compatibility; that would create semantic duplication.

## Architectural socket finding

Flutter `TicketWorkspaceScreen` currently uses ticket-scoped direct raw-socket listeners for escrow events because the singleton `SocketService` callback API does not yet provide independent ticket-scoped subscriptions. Replacing these listeners with a single callback would create consumer clobbering.

A future socket abstraction should support multiple subscription handles with event + ticket filtering, reconnect-safe registration and exact unsubscription while retaining the single underlying Socket.IO connection.

## Next priority order

1. Finish and verify Flutter PR #30 exact-head CI; merge only after its repository gate passes.
2. Fix `markSatisfied()` terminal-state race with a dedicated concurrency regression test after tracing the complete escrow release implementation and relevant database guards.
3. Resolve the order webhook semantic contract after tracing all producers and consumers across Backend, Business Portal, Admin Portal and Flutter.
4. Build cross-repository event-contract tests so backend event renames cannot silently break downstream convergence.
5. Add Business Portal realtime/query-bridge tests.
6. Verify customer order tracking end-to-end.
7. Then proceed to the remaining roadmap workstreams (marketplace verticals, business operating system, admin command center, observability).

## Session continuity / duplication rule

Before creating any implementation branch:

- read this file and the master roadmap;
- inspect current `main` in every affected repository;
- search existing event names, services, providers, routes and tests;
- inspect all open PRs and compare their heads against current `main`;
- prefer extending an existing canonical abstraction over creating a parallel one;
- if a previous PR is stale, close/rebase/replace it rather than implementing the same logical change again;
- after merge, re-audit the resulting `main` for duplicate code paths and stale branches.

A green CI run is necessary but not sufficient: concurrency semantics, transaction boundaries, event ordering, and cross-repo consumer coverage must also be verified.
