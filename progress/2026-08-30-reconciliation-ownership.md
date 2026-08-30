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

Backend #40 was superseded because its branch was based on an older mainline. Backend #41 is the replacement mainline-based tracking hardening batch and is the only active backend tracking PR. It centralizes customer/business-owner authorization for tracking reads, protects the timeline endpoint, preserves legitimate zero-valued telemetry, validates courier and delivery coordinates, validates ETA input, and uses one authoritative timestamp for persisted status timeline, actual arrival, and emitted status events.

The tracking contract remains intentionally additive: existing routes and Socket.IO event names are unchanged. No second socket layer, tracking state machine, or financial mutation was introduced.

## Business Portal realtime lifecycle

Business Portal #8 added the existing realtime-to-query convergence bridge and Business Portal #8 was merged. A follow-up lifecycle audit found that a socket destroyed during logout could be replaced after login without the query invalidation bridge being rebound. Business Portal #8's follow-up lifecycle fix is merged as part of the current mainline state.

The invariant is:

`authenticated session → current socket → one realtime query bridge → canonical query refetch`

Socket payloads remain invalidation signals rather than a competing source of truth.

## Flutter tracking lifecycle audit

Flutter uses a singleton SocketService with one connection, listener registry, and reconnectable room registry. The existing OrderTrackingScreen previously replaced the singleton order callbacks with no-op callbacks during disposal. That could clear callbacks belonging to a newer screen/provider.

Flutter PR #19 addresses this by giving the tracking screen stable callback identities, adding identity-based removal to the singleton, guarding queued post-frame registration with `mounted`, and ignoring tracking events whose `orderId` does not match the active screen.

The implementation deliberately keeps the existing socket, order room and event names. It does not introduce a second realtime abstraction.

## Cross-system contract

The operational path is now:

`financial/domain state → durable exception → authenticated Control Plane API → atomic operator ownership → audited action → realtime invalidation → canonical refetch`

For order tracking the same principle applies:

`authoritative order/tracking mutation → existing Socket.IO event → client invalidation/refetch`

The server remains authoritative for identity, authorization, coordinates, ETA and tracking state.

## Next deep audit

After the active backend and Flutter gates are green, audit the existing backend emitters against the Business Portal and Flutter consumers for the competition-critical domains:

1. escrow lifecycle;
2. orders/checkout/tracking;
3. invoices;
4. reservations/transit/Dine-In;
5. notifications and cache invalidation.

For each event, identify the authoritative backend mutation, emitted event name/payload, client listener, query/cache invalidation, reconnect behavior, and duplicate-listener risk. Implement missing contracts in coherent cross-repository batches rather than per-screen patches.

## Guardrails

Do not add a second socket abstraction, second financial source of truth, or generic Admin resolution that merely changes exception status without evidence-backed domain action.

Do not merge a client contract against an unverified backend head.

Before merging any batch, re-audit the complete PR diff against the target branch for duplicated implementations, stale branches, accidental formatting churn, and event-name drift.
