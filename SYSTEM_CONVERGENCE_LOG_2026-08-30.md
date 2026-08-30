# System Convergence Log — 2026-08-30

## Purpose

Record cross-repository findings that affect the authoritative state flow between Backend, Business Portal, Admin Portal and Flutter. This log is deliberately implementation-oriented: a green CI result is evidence for a commit, while the architectural conclusions below are derived from tracing the actual source contracts.

## Batch: invoice payment realtime convergence

### Status

**IMPLEMENTED / VERIFIED / MERGED**

Business Portal PR #10 was merged after its Business Portal CI run completed successfully.

Merge commit: `047d5891195060374e25749401e0ec032dfa3b7d`

### Research trace

The affected path was traced before implementation:

1. `services/businessInvoiceService.js` — `payInvoice()` performs the customer debit, business credit, fee accounting, invoice `PAID` transition and transaction-history rows inside one transaction.
2. `controllers/businessInvoiceController.js` — after the transaction commits, the controller emits `invoice_paid` to the business owner's `user_<id>` socket room and sends the existing business-owner notification.
3. `src/lib/socket.js` — Business Portal owns one Socket.IO singleton and authenticates the handshake with the current token.
4. `src/lib/realtimeQueryBridge.js` — the existing singleton-per-socket convergence boundary maps socket events to React Query invalidation.
5. `src/pages/Invoices.jsx` — the business invoice console uses `biz-invoices`, `biz-invoice`, and `invoice-stats` query keys.

### Finding

`invoice_paid` was already subscribed by the Business Portal convergence bridge, but it was grouped with order/escrow events and therefore called `invalidateOrder()`. That refreshed order queries while failing to invalidate the canonical invoice queries used by the invoice console.

This was a consumer contract bug, not a missing backend event.

### Implementation

The existing realtime bridge was extended rather than creating a second transport or state store:

- extract `invoiceId` from `invoice_paid` payloads;
- invalidate `['biz-invoice', invoiceId]` and the legacy-compatible raw-id form;
- invalidate `['biz-invoices']`;
- invalidate `['invoice-stats']`;
- keep order invalidation for order/tracking/escrow events only;
- preserve the existing singleton-per-socket listener ownership model.

### Resulting authoritative flow

```text
Business invoice transaction commits
        ↓
backend emits invoice_paid
        ↓
Business Portal receives notification
        ↓
invoice queries invalidated
        ↓
canonical invoice API refetch
        ↓
PAID state rendered from backend truth
```

No invoice state is copied from the socket payload into client state.

## Related escrow concurrency audit

A previously suspected `SmartEscrow.markSatisfied()` stale-state race was re-researched against the current database architecture before any new implementation was attempted.

The current database integrity layer already installs `azm_guard_smart_escrow_funding_transition`. Its `BEFORE UPDATE` trigger rejects a transition into `PENDING_SETTLEMENT` when the persisted escrow is already `SETTLED`, `RELEASED`, `REFUNDED`, or `EXPIRED`. The same guard also rejects invalid duplicate `DRAFT → FUNDED` transitions.

Therefore the previously described race cannot silently overwrite a terminal escrow state in the current database. The remaining concern is request-level UX: a losing concurrent caller may receive a transition error after another caller has successfully settled. That is a separate service/API response-convergence concern and must not be confused with a persisted-state regression.

The existing guard is evidence that the platform's current design already prefers database-boundary state invariants for this class of race. Do not introduce a duplicate trigger or second state machine.

## Cross-portal event contract notes

- Order tracking uses canonical colon-delimited events: `order:location`, `order:status`, `order:eta`.
- Business Portal retains legacy underscore aliases only for rolling-deployment compatibility.
- Business owner escrow notifications are represented through the existing business notification channel and the established `ORDER_*` notification types.
- `invoice_paid` is a business-owner event and must converge invoice resources, not order resources.
- Admin realtime remains a separate control-plane channel (`admin_spy_room`) and should not be merged into the business/customer socket contract.

## Duplication audit

No new socket connection, event bus, query cache, invoice store, escrow service, or parallel notification abstraction was introduced by this batch.

The implementation extends the existing convergence boundary and keeps HTTP/API responses authoritative.

## Next high-value work

Trace the financial notification/event matrix across:

**Backend financial mutation → Business Portal → Admin control plane → Flutter customer/business experience**

For each event, classify:

- authoritative state resource;
- producer;
- socket room;
- event name;
- consumer;
- canonical query/state invalidation;
- idempotency behavior;
- duplicate delivery behavior;
- reconnect behavior;
- terminal-state protection.

Do not create new event names until this matrix proves the existing contract is insufficient.
