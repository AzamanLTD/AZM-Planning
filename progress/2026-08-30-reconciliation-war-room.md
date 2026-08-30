# Reconciliation War-Room Batch — 2026-08-30

## Completed foundation

Backend #35 is merged. `Withdrawal.transactionHistoryId` is now the durable identity bridge to the canonical `TransactionHistory`; legacy reconciliation refuses to guess when the bridge is absent or ambiguous.

Backend #34 is merged. Unsafe reconciliation outcomes are durable `ReconciliationException` records rather than silent skips.

## Current implementation

Backend #36 exposes the operational exception queue through the existing Admin Control Plane boundary:

- `GET /api/admin/control-plane/exceptions` — paginated OPEN/RESOLVED/ALL queue.
- `POST /api/admin/control-plane/exceptions/:id/resolve` — protected resolution with mandatory reason.
- Resolution records `StaffActivityEvent` for auditability.
- Financial state is never changed by exception-management endpoints.

Admin Portal #11 consumes those endpoints inside the existing Control Plane page rather than creating a competing war-room implementation.

The Admin Portal uses its existing singleton realtime connection. `reconciliation_exception` only invalidates the React Query projection; it never patches financial values from socket payloads.

## Verification policy

Both Backend #36 and Admin #11 remain open until their CI gates are green. After CI, the complete PR diffs must be re-read for duplicated route mounts, duplicated realtime listeners, and unrelated changes before merge.

## Next system boundary

After this batch is verified, the reconciliation queue should evolve from visibility to controlled resolution workflows where appropriate, with explicit domain-specific resolution actions rather than a generic status toggle. The same event/invalidation contract should then be audited across Flutter escrow/order/invoice flows and Business Portal invoice/reservation/transit/Dine-In projections.

## Architectural invariant

`domain state → durable exception → authenticated operational API → audited operator action → realtime invalidation → canonical refetch`

The exception queue is an operational projection. It is never the financial source of truth.
