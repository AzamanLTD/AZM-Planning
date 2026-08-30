# System convergence progress — 2026-08-30

## Completed

### Backend
- Order tracking reads now enforce customer/business-owner object authorization.
- Courier telemetry validates finite latitude/longitude, heading and speed values.
- Legitimate zero coordinates are preserved.
- Delivery coordinates are validated.
- Tracking timeline, `actualArrival`, and `order:status` share one authoritative event timestamp.
- These changes are on `main` after the backend test gate passed.

### Business Portal
- Existing singleton realtime bridge now consumes the backend's canonical `order:location`, `order:status`, and `order:eta` events.
- Existing underscore aliases remain temporarily for rolling-deployment compatibility.
- `invoice_paid` is included in the existing escrow/order invalidation group.
- Socket replacement after logout/login re-establishes the existing query-convergence bridge.
- The bridge remains notification-only; canonical HTTP/React Query data remains the source of truth.

### Flutter
- Tracking screen lifecycle was audited against the singleton `SocketService`.
- The tracking screen no longer removes unrelated singleton listeners during disposal.
- Tracking callbacks are scoped to the active order and reconnectable room ownership remains centralized in `SocketService`.

## New escrow findings

### Business no-show penalty outcome
The existing `penaltyPolicyService` previously returned `penaltyApplied: true` even when the business stake was too small for the configured penalty and no deduction occurred. This was corrected in Backend PR #42 with focused tests for:

1. successful penalty deduction;
2. insufficient stake;
3. failed escrow refund.

The returned `penaltyAmount` and `penaltyApplied` values are now also included in the audit metadata.

### Satisfaction transition race — next hardening target
The existing `markSatisfied` flow atomically claims each party's satisfaction flag, but the non-settling branch performs an unconditional `status = PENDING_SETTLEMENT` update after reading the flags. A concurrent second participant can settle the escrow between that read and the unconditional write, allowing the first request to overwrite `SETTLED` with `PENDING_SETTLEMENT` after the financial release has already happened.

The next backend hardening batch must make that final status transition conditional on the escrow still being in a non-final state and add a regression test for the interleaving.

## Architectural rule

The client portals must not become competing financial or order state stores. Backend mutations remain authoritative. Socket events are convergence signals; clients invalidate/refetch canonical representations. Financial transitions remain atomic and idempotent in the backend.

## Open implementation

- Backend PR #42: business penalty outcome correctness — awaiting CI.
- Backend branch `fix/escrow-satisfaction-race`: research complete for the next satisfaction race fix; implementation is intentionally kept separate so the two financial correctness changes remain independently reviewable.
