# Financial Event Contract Matrix — 2026-08-30

## Purpose

This document is a cross-repository implementation map for financial and order-state propagation. It is intentionally based on source/commit research already verified in the AZM repositories. It is not a proposal to invent a second event system.

The governing rule is:

> Socket events are convergence signals. The backend/database/API remains authoritative.

A client receiving an event should invalidate/refetch or otherwise reconcile against the canonical resource rather than treating an event payload as an independently authoritative copy of state.

## Verified producer contracts

### Escrow lifecycle

The backend escrow implementation already has persistent business-owner notification events covering:

- `ORDER_FUNDED`
- `ORDER_SATISFIED`
- `ORDER_DISPUTED`
- `ORDER_SETTLED`
- `ORDER_CANCELLED`
- `ORDER_REFUNDED`

These are persisted through the existing `BusinessNotification` system. The producer implementation uses fire-and-forget notification work after/around the relevant financial transition so notification delivery does not become the financial transaction's success condition.

The backend also synchronizes the linked `BusinessOrder` from escrow transitions, including FUNDED → PAID, SETTLED/RELEASED → COMPLETED, REFUNDED → REFUNDED, and DISPUTED → DISPUTED.

### Invoice payment

`invoice_paid` is an existing backend socket event emitted after the invoice payment transaction commits. It is a business-owner event.

The Business Portal convergence boundary now maps it to invoice resources rather than order resources.

Canonical Business Portal query families:

- `biz-invoice/:id`
- `biz-invoices`
- `invoice-stats`

### Order tracking

The canonical tracking socket event vocabulary is:

- `order:location`
- `order:status`
- `order:eta`

The Business Portal retains older underscore aliases only as rolling-deployment compatibility. New backend/client code must prefer the colon-delimited contract.

## Consumer matrix

| Domain event | Authoritative backend resource | Business Portal | Admin Portal | Flutter |
| --- | --- | --- | --- | --- |
| `order:location` | BusinessOrder/tracking state | invalidate order detail/list/stats | control-plane visibility if subscribed | update/reconcile active order tracking |
| `order:status` | BusinessOrder/order lifecycle | invalidate order detail/list/stats | control-plane visibility if subscribed | reconcile active order |
| `order:eta` | tracking/ETA state | invalidate order detail/list/stats | control-plane visibility if subscribed | reconcile active order |
| `invoice_paid` | Invoice + transaction history | invalidate invoice detail/list/stats | audit/control-plane surface | only if customer invoice surface consumes it |
| `ORDER_FUNDED` | SmartEscrow + BusinessOrder | business notification + order convergence | admin control-plane/audit | customer order/escrow reconciliation |
| `ORDER_SATISFIED` | SmartEscrow | business notification + order convergence | admin control-plane/audit | customer escrow/order reconciliation |
| `ORDER_DISPUTED` | SmartEscrow + dispute | business notification + order convergence | dispute control-plane | customer escrow/order reconciliation |
| `ORDER_SETTLED` | SmartEscrow + ledger | business notification + order convergence | control-plane/audit | customer escrow/order reconciliation |
| `ORDER_REFUNDED` | SmartEscrow + ledger | business notification + order convergence | control-plane/audit | customer escrow/order reconciliation |
| `ORDER_CANCELLED` | SmartEscrow/BusinessOrder | business notification + order convergence | control-plane/audit | customer order/escrow reconciliation |
| `deposit_success` | TransactionHistory + user balance | only if business-owned deposit surface consumes it | financial oversight | balance/deposit reconciliation |

The Admin Portal must not be forced onto the business/customer socket contract. Existing architecture identifies `admin_spy_room` as the control-plane realtime channel.

## Important transport finding — escrow business notifications

The backend `bizNotificationService.notifyOrderEvent()` currently persists the `BusinessNotification` row but does not itself emit a Socket.IO event. The general-purpose `NotificationService` does have a DB + Socket.IO + FCM pipeline, but it is a separate user-notification model and is not the BusinessNotification feed.

This means the escrow hooks in `escrowService` can successfully create `ORDER_FUNDED`, `ORDER_SATISFIED`, `ORDER_DISPUTED`, `ORDER_SETTLED`, and `ORDER_REFUNDED` records while the Business Portal receives no direct realtime signal from that persistence operation unless another controller/socket path emits one.

This is a confirmed transport-gap candidate, not yet an implementation decision.

Before fixing it, research must identify:

1. the canonical Socket.IO instance ownership path;
2. the authorized business-owner room naming contract;
3. whether `biz_notification` is already emitted by any escrow/business-order path;
4. whether adding emission at `bizNotificationService` would create duplicate socket events for controller paths that already emit them;
5. whether the payload should carry the persisted notification record or only a minimal invalidation signal;
6. whether a multi-instance Socket.IO adapter is already authoritative for the target deployment.

Only after those checks should the transport gap be implemented. The preferred shape is one existing transport path with idempotent client invalidation, not another event bus.

## Idempotency and delivery rules

1. Financial mutations must remain protected by their existing transaction/idempotency boundaries.
2. Socket delivery is not a financial commit mechanism.
3. Duplicate socket delivery must be harmless because clients invalidate/refetch canonical resources.
4. Reconnect must restore the relevant socket room/listener set without accumulating duplicate listeners.
5. Terminal escrow state must remain protected at the database boundary. Do not add a second state machine merely to compensate for a client race.
6. A losing concurrent API request may legitimately receive a transition conflict after another request wins; client handling should reconcile from the authoritative resource rather than displaying a stale optimistic terminal state.

## Known implementation already merged

### Business Portal realtime/query convergence

The existing singleton-per-socket bridge now owns event-to-query invalidation. It handles canonical tracking events, escrow lifecycle events, business notifications, and `invoice_paid`.

The bridge deliberately does not copy event payloads into a parallel application state store.

### Business Portal authentication/socket lifecycle

The realtime bridge is explicitly re-installed when authentication creates/restores a socket so a logout/login socket replacement cannot leave the new socket without convergence listeners.

### Flutter socket lifecycle

Flutter's `SocketService` is a singleton with reconnectable room registration. The order tracking screen was hardened so disposing one screen cannot replace shared singleton callbacks with no-op handlers belonging to another consumer.

## Research findings that must prevent duplication

### Do not add another escrow transition guard

The backend already has a database trigger that blocks stale terminal-state regressions into `PENDING_SETTLEMENT`, including attempts against `SETTLED`, `RELEASED`, `REFUNDED`, and `EXPIRED` escrows.

Therefore a second trigger or independent escrow state machine would duplicate an existing invariant.

### Do not create another notification bus

The backend already has `BusinessNotification` persistence and business-owner `ORDER_*` notification types. Existing socket notifications and persisted notifications are complementary delivery surfaces, not justification for a third event mechanism.

### Do not make socket payloads authoritative

The Business Portal convergence implementation establishes the desired pattern: event → invalidate → canonical API refetch.

Flutter should follow the same principle for order/escrow state while retaining its platform-specific UI state.

## Next implementation sequence

### Batch 1 — backend producer inventory

For each financial mutation, record the exact event name, room, payload identifier, transaction boundary, and persisted notification/audit record.

### Batch 2 — Business Portal

Verify every producer event has exactly one convergence mapping and that the query key invalidated is the resource actually changed by the event.

### Batch 3 — Flutter

Verify every customer-visible financial event has one lifecycle-safe consumer and that reconnect/duplicate delivery cannot create stale state.

### Batch 4 — Admin

Trace the control-plane event channel independently. Do not merge admin oversight traffic into customer/business event namespaces without a concrete contract requirement.

### Batch 5 — end-to-end regression

For each event, test:

- producer commits authoritative state;
- event emitted only after the relevant mutation boundary;
- correct audience/room;
- duplicate delivery;
- reconnect;
- stale event ordering;
- canonical refetch;
- terminal-state consistency;
- notification/audit persistence.

## Definition of done

The financial event surface is complete when an event can be traced from its authoritative mutation to every authorized consumer, with no unexplained event name, no orphaned consumer, no duplicate state machine, and no client path that can remain permanently stale after reconnect.
