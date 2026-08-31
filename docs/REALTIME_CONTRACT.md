# AZM Cross-Portal Realtime Contract

Last reviewed: 2026-08-31

This document is an architecture guardrail for the Backend, Admin Portal, Business Portal, and Flutter app. It is not a second implementation of realtime behavior. Backend events remain the authoritative notification mechanism; clients must converge by refetching canonical API state where financial/order state is involved.

## Rules

1. One Socket.IO connection per client runtime. Do not introduce a second socket to solve a missing subscription.
2. Backend commits the authoritative mutation before emitting a state-change event.
3. Event payloads are hints/invalidation signals, not authoritative financial balances, invoice status, order status, or escrow state.
4. A client consumer must be idempotent. Repeated delivery must produce the same canonical projection and must not perform a second financial mutation.
5. Listener registration must have a matching, exact unsubscribe path. Never remove another component's listener with a broad `off(event)`.
6. Before adding an event, search all five repositories for existing producers and consumers. If an equivalent event already exists, extend its canonical consumer rather than creating a duplicate event family.
7. After every realtime change, search mainline again for duplicate socket connections, duplicate event names, duplicate invalidation handlers, and direct client-side financial mutations.

## Verified event boundaries

| Event | Backend meaning | Canonical client response | Duplicate-risk rule |
|---|---|---|---|
| `invoice_received` | Business invoice has been created/sent to the customer | Flutter invalidates/refetches customer invoices | Do not create a second invoice-created event |
| `invoice_paid` | Invoice payment committed | Business Portal and Flutter invalidate/refetch invoice state | Payload must not be treated as a balance source |
| `business_order_delivered` | Business order has committed `PAID → DELIVERED` | Customer/business order projections invalidate/refetch | Do not emit a second delivered event for the same transition |
| Escrow lifecycle events | Escrow mutation committed | Ticket/escrow projections invalidate/refetch | One canonical event per committed transition |

## Order webhook boundary

External webhooks and internal Socket.IO events are separate contracts. The current codebase has a semantic boundary that must be preserved:

- `PAID → DELIVERED` is a delivery transition.
- `DELIVERED → COMPLETED` is a completion transition.
- The external webhook named `order.completed` must only represent the latter transition.

Before changing this contract, audit every backend producer, Business Portal consumer/configuration, Admin Portal consumer, documentation, tests, and any external-facing webhook schema. Do not solve a semantic mismatch by emitting both names for one transition.

## Escrow concurrency boundary

`markSatisfied` is already protected by database-level terminal-state rules. The remaining convergence requirement is:

```text
concurrent satisfaction
       ↓
one transaction wins settlement
       ↓
losing request observes committed terminal state
       ↓
losing request returns canonical state
       ↓
no second settlement / ledger mutation
```

A fix must not weaken the database guard or introduce a second settlement path.

## Ticket-scoped subscriptions

The Flutter Ticket Workspace has a legitimate need for multiple ticket-specific escrow subscriptions while the singleton socket service owns global dispatch. Replacing ticket listeners with one global callback would create last-subscriber-wins behavior.

The canonical future design is a subscription registry with:

- event filtering;
- ticket filtering;
- per-subscription unsubscribe handles;
- reconnect-safe re-registration;
- widget lifecycle cleanup;
- one underlying socket.

Do not implement this by adding another Socket.IO connection.

## Verification checklist

For every realtime implementation:

- [ ] Search all repositories for the event name before implementation.
- [ ] Identify every producer and consumer.
- [ ] Identify every existing socket connection and singleton dispatcher involved.
- [ ] Confirm backend mutation commits before emission.
- [ ] Confirm client reaction is canonical refetch/invalidation where appropriate.
- [ ] Add duplicate-delivery regression coverage.
- [ ] Add reconnect/unsubscribe coverage where subscriptions are involved.
- [ ] Run exact-head CI.
- [ ] Re-audit resulting mainline for duplicate events/listeners/connections.
- [ ] Update this contract and `MASTER_ARCHITECTURE_AND_EXECUTION_STATE.md` with the resulting decision.
