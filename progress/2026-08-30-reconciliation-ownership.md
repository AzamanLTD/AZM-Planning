# Reconciliation Ownership Batch — 2026-08-30

## Verified foundation

Backend #37 established monotonic provider-attempt terminal state: late contradictory callbacks cannot regress the provider evidence record after a terminal outcome.

Backend #39 is now merged. Reconciliation exceptions support atomic 15-minute ownership leases with claim, release, expiry-based reclaim, and owner-constrained resolution. Claim/release/resolve actions remain operational only and do not mutate balances, ledger records, provider attempts, or provider state.

Admin Portal #12 is now merged and consumes the same ownership API through the existing Control Plane and existing realtime/query architecture.

## Ownership invariant

`OPEN + unclaimed → claimable`

`OPEN + active owner → only owner may release/resolve`

`OPEN + expired owner → reclaimable`

`RESOLVED → terminal operational state`

The lease is concurrency-controlled in the database. Client-side state is never treated as authority.

## Tracking integrity audit

Backend #40 is the active order-tracking hardening batch. Its existing changes centralize customer/business-owner authorization for tracking reads, protect the timeline endpoint, preserve legitimate zero-valued telemetry, and validate ETA input. The branch was re-researched before extending it with numeric/range validation for latitude, longitude, heading, and speed. Invalid telemetry is rejected before database access.

The tracking contract remains intentionally additive: existing routes and Socket.IO event names are unchanged. No second socket layer, tracking state machine, or financial mutation was introduced.

## Cross-system contract

The operational path is now:

`financial/domain state → durable exception → authenticated Control Plane API → atomic operator ownership → audited action → realtime invalidation → canonical refetch`

Realtime events remain invalidation signals. The Admin Portal must refetch authoritative exception state after claim/release/resolve rather than trusting socket payloads.

For order tracking the same principle applies:

`authoritative order/tracking mutation → existing Socket.IO event → client invalidation/refetch`

The server remains authoritative for identity, authorization, coordinates, ETA and tracking state.

## Realtime lifecycle finding

The merged Business Portal realtime bridge correctly treats socket payloads as invalidation signals and replaces listeners when the socket identity changes. The audit then identified a lifecycle gap: the initial bootstrap stops after the first socket is bound, while logout destroys that socket and a later login creates a new singleton instance. Without an explicit rebind hook, the new session can be connected to realtime transport but lack the query invalidation bridge.

Business Portal #8 addresses this by exposing an idempotent `ensureRealtimeQueryBridge()` hook from the existing query client and invoking it whenever AuthContext wires a restored or newly logged-in session socket. This extends the existing bridge; it does not create another socket or event layer.

Flutter research also confirmed a separate lifecycle hazard in `OrderTrackingScreen`: the screen previously replaced the singleton order callbacks with no-op callbacks during `dispose()`. A first attempted fix was intentionally discarded because it rewrote too much unrelated formatting. The branch was reset and no partial Flutter implementation was retained. The next Flutter batch must preserve the original file and change only the callback ownership/lifecycle behavior with focused regression coverage.

## Next deep audit

Before adding more UI, audit the existing backend emitters against the Business Portal and Flutter consumers for the competition-critical domains:

1. escrow lifecycle;
2. orders/checkout/tracking;
3. invoices;
4. reservations/transit/Dine-In;
5. notifications and cache invalidation.

For each event, identify the authoritative backend mutation, emitted event name/payload, client listener, query/cache invalidation, reconnect behavior, and duplicate-listener risk. Implement missing contracts in coherent cross-repository batches rather than per-screen patches.

## Guardrail

Do not add a second socket abstraction, second financial source of truth, or generic Admin resolution that merely changes exception status without evidence-backed domain action.
