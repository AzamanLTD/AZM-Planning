# Financial Event Contract Matrix — 2026-08-30

## Purpose

Define the cross-repository contract before adding more realtime code. Backend remains authoritative; socket delivery is a convergence signal; clients refetch canonical state after accepted events.

## Repository roles

| Repository | Role | Must never become authoritative for |
|---|---|---|
| `AZM-backend` | financial/domain source of truth + event producer | — |
| `AZM-businessPortal` | merchant operational projection | balances, escrow, invoice truth |
| `AZM-adminPortal` | governance/control-plane projection | financial truth |
| `AZM-frontend` | consumer projection | financial/order truth |
| `AZM-planning` | engineering memory | runtime state |

## Event handling rules

1. A financial mutation commits authoritative database state before its domain event is emitted.
2. Event handlers must be idempotent at the client boundary.
3. An event must trigger invalidation/refetch, not become a second source of truth.
4. Event payloads must contain enough stable identity to target the affected resource (`eventId` where durable notification identity exists; otherwise a domain resource ID plus event name).
5. Terminal events must never permit a client to regress a known terminal state from a realtime payload.
6. Reconnect must restore required rooms and must not duplicate listener registration.
7. Legacy event aliases may be retained temporarily only when they are explicitly documented as rolling-deployment compatibility.
8. Each domain owns one canonical event vocabulary. Do not create portal-specific financial event names.

## Current verified client surface

### Flutter

`lib/services/socket_service.dart` owns the singleton Socket.IO connection and callback registry. It currently handles:

- `balance_update`
- `deposit_success`
- `withdrawal_progress`
- `withdrawal_settled`
- canonical tracking events `order:location`, `order:status`, `order:eta`
- temporary tracking aliases `order_location`, `order_status`, `order_eta`
- escrow-related events `escrow_funded`, `escrow_settled`, `escrow_pending_settlement`, `escrow_disputed`, `escrow_resolved`, `escrow_terms_updated`, `invoice_paid`

`RealtimeEventDeduper` is bounded and session-scoped. It currently protects persisted business notifications when a stable notification ID is present. Missing IDs are accepted rather than inventing unsafe identity semantics.

### Business Portal

The realtime query bridge is the convergence boundary for merchant-facing order/invoice state. The invoice-paid path was corrected so `invoice_paid` invalidates invoice-specific, invoice-list and invoice-stat queries rather than incorrectly treating an invoice event as an order-only event.

### Backend

Financial state remains authoritative in transactional services and database invariants. Existing escrow terminal-state protections must be reused rather than duplicated in application code or new triggers.

## Event matrix to implement/audit

| Domain transition | Canonical resource | Producer requirement | Business Portal | Admin Portal | Flutter |
|---|---|---|---|---|---|
| invoice paid | invoice | commit then emit `invoice_paid` | invalidate invoice detail/list/stats | audit/control projection | refresh relevant financial/order surface |
| escrow funded | escrow/order | commit then emit canonical escrow/order event | order convergence | governance/audit convergence | escrow/order convergence |
| escrow satisfied | escrow | persist satisfaction before notification | owner notification/convergence | audit where appropriate | participant state convergence |
| escrow settled | escrow/order | settlement winner + commit before event | order/finance convergence | financial governance/audit | order/escrow convergence |
| escrow disputed | escrow/dispute | dispute transaction then event | order/dispute convergence | dispute queue/convergence | escrow/dispute convergence |
| escrow resolved | escrow/dispute | authoritative ruling then event | order/finance convergence | dispute resolution view | escrow/finance convergence |
| escrow refunded | escrow/order | refund transaction then event | order/finance convergence | financial audit | escrow/finance convergence |
| withdrawal progress | withdrawal | durable state transition then event | merchant finance where applicable | financial operations | withdrawal UI |
| withdrawal settled | withdrawal | durable settlement then event | merchant finance where applicable | financial operations | withdrawal UI |
| balance mutation | user ledger projection | authoritative mutation then balance event | invalidate/refetch affected finance views | audit/projection | update/invalidate canonical balance |

## Required regression scenarios

### Delivery

- event delivered once;
- same event delivered twice;
- event delivered after reconnect;
- event delivered while screen/provider is being disposed;
- old socket replaced by a new authenticated socket.

### Ordering

- stale non-terminal event arrives after a newer event;
- terminal event arrives before an older event;
- two concurrent satisfaction calls;
- two concurrent settlement attempts.

### Financial safety

- duplicate client request;
- timeout followed by retry;
- provider callback retry;
- notification retry;
- insufficient balance/stake;
- failed downstream synchronization after committed financial state.

## Current known findings

- Backend escrow database protections already reject invalid terminal-state regressions; do not duplicate this with another trigger or parallel state machine.
- Flutter tracking listener removal was hardened to identity-based removal so one disposed screen cannot clear another consumer's singleton callback.
- Business Portal tracking event names were aligned with the backend canonical colon-delimited contract, while legacy aliases remain only for rolling compatibility.
- Business Portal invoice convergence now uses the invoice query family.
- Backend no-show penalty reporting was corrected so `penaltyApplied` reflects the actual ledger outcome rather than the requested action.

## Implementation order

1. Finish producer inventory in backend by domain and exact emitted event/payload.
2. Map every consumer against that inventory.
3. Remove or isolate mismatched/duplicate client event names.
4. Add contract tests around stable identity and terminal ordering.
5. Add client convergence tests for duplicate/reconnect delivery.
6. Only then add new portal UI behavior.

## Exit criteria

The matrix is complete when every P0 financial transition has a verified producer, payload identity, authorized recipient scope, Business Portal consumer, Admin consumer where applicable, Flutter consumer where applicable, canonical refetch target, duplicate-delivery behavior, reconnect behavior and regression test.
