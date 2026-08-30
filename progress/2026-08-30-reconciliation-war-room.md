# Reconciliation War-Room Batch — 2026-08-30

## Completed foundation

Backend #35 is merged. `Withdrawal.transactionHistoryId` is now the durable identity bridge to the canonical `TransactionHistory`; legacy reconciliation refuses to guess when the bridge is absent or ambiguous.

Backend #34 is merged. Unsafe reconciliation outcomes are durable `ReconciliationException` records rather than silent skips.

Backend #37 is merged. Provider settlement attempts now preserve terminal evidence against contradictory late callbacks and return the database-winning terminal state during concurrent callback races.

## Current implementation

Backend #36 exposes the operational exception queue through the existing Admin Control Plane boundary:

- `GET /api/admin/control-plane/exceptions` — paginated OPEN/RESOLVED/ALL queue.
- `POST /api/admin/control-plane/exceptions/:id/resolve` — protected resolution with mandatory reason.
- Resolution records `StaffActivityEvent` for auditability.
- Financial state is never changed by exception-management endpoints.

Backend #39 extends that queue with controlled operator ownership:

- `POST /api/admin/control-plane/exceptions/:id/claim` — atomically acquires a 15-minute lease for an authenticated staff operator.
- `POST /api/admin/control-plane/exceptions/:id/release` — owner releases the lease; expired leases may be released safely.
- Resolution is constrained by active ownership and may only proceed for the owner or an expired lease.
- Claim and release actions are audited through the existing `StaffActivityEvent` path.
- Ownership is stored as operational metadata on the exception; it never becomes financial state.

Admin Portal #11 consumes the queue inside the existing Control Plane. Admin Portal #12 extends the same component/API client with claim, release, lease-expiry display, and ownership-aware resolution. No second realtime connection or competing queue was introduced.

The Admin Portal uses its existing singleton realtime connection. `reconciliation_exception` only invalidates the React Query projection; it never patches financial values from socket payloads.

## Verification policy

Backend #37 passed the full backend gate before merge. Backend #39 and Admin #12 must pass their complete CI gates before merge. After CI, the complete PR diffs must be re-read for duplicated route mounts, duplicated realtime listeners, stale base branches, and unrelated changes before merge.

## Next system boundary

After the ownership workflow is verified, reconciliation should evolve from generic resolution toward domain-specific, evidence-backed actions where appropriate. In parallel, audit the realtime/event contracts across Flutter escrow/order/invoice flows and Business Portal invoice/reservation/transit/Dine-In projections.

## Architectural invariant

`domain state → durable exception → authenticated operational API → audited operator ownership/action → realtime invalidation → canonical refetch`

The exception queue is an operational projection. It is never the financial source of truth.
