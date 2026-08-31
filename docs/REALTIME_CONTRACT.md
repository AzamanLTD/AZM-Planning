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
8. Static `.emit()`/`.on()` name comparison is not sufficient. Audit conditional/dynamic event names and callbacks before classifying an event as orphaned.

## Verified event boundaries

| Event | Backend meaning | Canonical client response | Status |
|---|---|---|---|
| `invoice_received` | Business invoice has been created/sent to the customer | Flutter invalidates/refetches customer invoices | Verified existing contract |
| `invoice_paid` | Invoice payment committed | Business Portal and Flutter invalidate/refetch invoice state | Verified existing contract |
| `business_order_delivered` | Business order has committed `PAID → DELIVERED` | Customer/business order projections invalidate/refetch | Verified existing contract |
| `escrow_funded` | Escrow funding transaction committed | Escrow/ticket projections converge from canonical state | Backend producer added and regression-tested in PR #58 |
| `escrow_pending_settlement` | One party satisfied; escrow remains pending the other party | Escrow/ticket projections converge from canonical state | Backend producer added and regression-tested in PR #58 |
| `escrow_settled` | Escrow settlement transaction committed with terminal `SETTLED` state | Escrow/ticket projections converge from canonical state | Backend producer added and regression-tested in PR #58 |
| `escrow_refunded` | Canonical refund transaction committed | Escrow/ticket projections converge from canonical state | Existing canonical producer verified |
| `escrow_disputed` | Escrow dispute transition committed | Escrow/ticket projections converge from canonical state | Existing producer/consumers verified |
| `escrow_resolved` | Dispute resolution transition committed | Escrow/ticket projections converge from canonical state | Existing producer/consumers verified |
| `escrow_terms_updated` | Delivery terms successfully updated | Escrow/ticket projections converge from canonical state | Existing producer in `escrowController.updateTerms` verified |
| `trade_update` | P2P trade state update; completion is represented by `status: COMPLETED` | Active trade projections react to the status and converge from canonical state | Canonical completion event |
| `group_typing_started` | Group member began typing | Group-chat typing indicator updates | Existing dynamic producer verified |
| `group_typing_stopped` | Group member stopped typing | Group-chat typing indicator updates | Existing dynamic producer verified |
| `friend_transfer_received` | Friend transfer received | Flutter friend-transfer state updates | Existing consumer verified |
| `queue_position_update` | Queue position changed | Waiting-room queue projection updates | Existing consumer verified |
| `queue_promoted` | User promoted from queue into active trade | Waiting-room flow enters active trade | Existing consumer verified |
| `queue_update` | Queue state changed | Waiting-room queue projection updates | Existing consumer verified |

## Escrow lifecycle boundary

The canonical escrow lifecycle emits are owned by the financial service after the authoritative transaction commits:

```text
fund transaction commits
        ↓
escrow_funded

one party satisfies
        ↓
transaction commits
        ↓
escrow_pending_settlement

both parties satisfy / release path
        ↓
settlement transaction commits
        ↓
escrow_settled (SETTLED only)
```

`escrow_terms_updated` is intentionally separate: it is produced by `escrowController.updateTerms` after the `deliveryTerms` update succeeds. It is not a phantom listener and must not be replaced with another producer.

The dispute `RELEASED` path is represented by the existing `escrow_resolved` contract; `escrow_settled` must not be emitted for that status.

## Trade completion boundary

`trade_completed` is not a Backend producer and must not be introduced as a competing event. The canonical Backend completion signal is `trade_update` with `status: COMPLETED`, emitted after `completeTrade()` commits. The Flutter `trade_completed` callback/listener was dead code and was removed in the realtime cleanup PR.

## Group typing boundary

Group typing uses a two-stage protocol:

```text
client → group_typing { groupId, userId, isTyping }
                         ↓
Backend selects event name dynamically
                         ↓
true  → group_typing_started { groupId, userId }
false → group_typing_stopped { groupId, userId }
```

Therefore `group_typing_started` and `group_typing_stopped` are canonical emitted events. Do not rename them to a single `group_typing` consumer event unless the producer and every client are deliberately migrated together.

## Order event boundary

The canonical tracking event names are colon-delimited:

- `order:location`
- `order:status`
- `order:eta`

The Flutter underscore aliases `order_location`, `order_status`, and `order_eta` were dead duplicate listeners and were removed. New consumers must use the colon-delimited names.

External webhooks and internal Socket.IO events are separate contracts. The current codebase has a semantic boundary that must be preserved:

- `PAID → DELIVERED` is a delivery transition.
- `DELIVERED → COMPLETED` is a completion transition.
- The external webhook named `order.completed` must only represent the latter transition.

Before changing this contract, audit every backend producer, Business Portal consumer/configuration, Admin Portal consumer, documentation, tests, and any external-facing webhook schema. Do not solve a semantic mismatch by emitting both names for one transition.

## Escrow concurrency boundary

`markSatisfied` is protected by database-level terminal-state rules. Concurrent satisfaction must converge as follows:

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

## Orphan-event audit policy

The previous broad audit identified a large set of apparent Backend-only emits. Those are **not automatically dead**. Several false positives were established during verification because event names can be selected dynamically or code-search indexes can omit consumers.

Every apparent orphan must therefore be classified using runtime-aware evidence:

1. Resolve literal and dynamic producer event names.
2. Search all three client repositories for exact consumers and callback registration.
3. Identify internal/admin-only destinations before calling an event orphaned.
4. Classify as one of:
   - missing consumer;
   - missing producer;
   - intentional internal/admin event;
   - future/feature-specific event;
   - dead producer/consumer.
5. Remove only events proven dead, with focused regression coverage.

Verified examples that must **not** be removed as orphans include `group_typing_started`, `group_typing_stopped`, `friend_transfer_received`, and the queue event family.

## Verification checklist

For every realtime implementation:

- [ ] Search all repositories for the event name before implementation.
- [ ] Resolve dynamic/conditional event names, not only literal strings.
- [ ] Identify every producer and consumer.
- [ ] Identify every existing socket connection and singleton dispatcher involved.
- [ ] Confirm backend mutation commits before emission.
- [ ] Confirm client reaction is canonical refetch/invalidation where appropriate.
- [ ] Add duplicate-delivery regression coverage.
- [ ] Add reconnect/unsubscribe coverage where subscriptions are involved.
- [ ] Run exact-head CI.
- [ ] Re-audit resulting mainline for duplicate events/listeners/connections.
- [ ] Update this contract and `MASTER_ARCHITECTURE_AND_EXECUTION_STATE.md` with the resulting decision.
