# Realtime Convergence — 2026-08-30 Implementation Cycle

**Status:** IN PROGRESS / VERIFIED BATCHES RECORDED

This document records the source-derived implementation and verification work completed during the 2026-08-30 cross-repository realtime/convergence hardening cycle. It is intentionally additive: previous planning history is not rewritten.

## 1. System boundary

The current target architecture remains:

```text
Backend authoritative mutation
        ↓
post-commit domain signal / persisted notification
        ↓
existing authenticated Socket.IO transport
        ↓
client invalidation / UI signal
        ↓
canonical API read
        ↓
current projection
```

Socket payloads are not treated as a second financial source of truth. This rule is now implemented in the Business Portal and Admin Portal convergence boundaries and is preserved in Flutter's existing socket architecture.

## 2. Verified backend work

### Tracking hardening — Backend #41

The merged tracking batch centralizes participant authorization for tracking and timeline reads, validates finite/range-safe courier telemetry, preserves legitimate zero-valued coordinates/telemetry, validates ETA/delivery coordinates, and reuses one authoritative event timestamp for the persisted timeline, `actualArrival`, and `order:status` signal.

The replacement PR was based directly on current `main` after the older superseded branch was rejected as stale. The complete backend gate passed before merge.

### Escrow penalty outcome — Backend #42

The business no-show penalty path previously reported a penalty as applied even when the business did not have enough stake for an actual deduction. The merged correction reports the actual `penaltyApplied` and `penaltyAmount` outcome and records the result in audit metadata. Focused tests cover successful deduction, insufficient stake and failed refund behavior.

### Business notification transport — Backend #43

The existing BusinessNotification persistence chokepoint now emits `biz_notification` only after the notification row has successfully persisted. Delivery is best-effort and targets the existing authenticated user room. No second event bus or socket server was introduced.

This closes the backend persistence → realtime convergence gap for persisted business order notifications.

## 3. Business Portal convergence

### Socket lifecycle — Business Portal #8

The existing query bridge was made explicitly re-bindable when authentication replaces the singleton socket instance. Restore/login paths call the existing bridge bootstrap; socket identity checks remain idempotent and old listeners are removed before a replacement socket is bound.

### Event vocabulary — Business Portal #9

The bridge now consumes the backend's canonical tracking events (`order:location`, `order:status`, `order:eta`) while temporarily retaining underscore aliases for rolling-deployment compatibility. `invoice_paid` was placed on the invoice query invalidation path.

### Invoice convergence — Business Portal #10

`invoice_paid` now invalidates the canonical invoice detail/list/statistics query surfaces rather than the order projection. This follows the actual backend producer and the invoice console's canonical HTTP query keys.

### Order notification convergence — Business Portal #11

Persisted `ORDER_*` business notifications now converge both the notification feed and the affected order projection. The implementation remains on the existing `biz_notification` transport and canonical HTTP refetch model.

## 4. Admin Portal convergence

### Auth lifecycle — Admin Portal #13

The existing privileged singleton socket now rotates its Socket.IO auth token when session restoration produces a fresh JWT and is explicitly disconnected on logout/auth invalidation.

### Financial realtime — Admin Portal #14

The existing `useAdminRealtime` hook now invalidates the existing financial, escrow, order and reconciliation query surfaces from the admin socket. It deliberately does not patch cached financial values from socket payloads.

The backend `join_admin_spy` room and existing admin socket were retained; no second transport was introduced.

## 5. Flutter convergence

### Tracking lifecycle — Frontend #19 / #21 lineage

The singleton `SocketService` and tracking screen were hardened so a disposed tracking screen cannot remove another screen's callbacks. Order tracking callbacks are removed by identity, delayed registration is guarded, and events are checked against the active order identity.

### Persisted notification deduplication — Frontend #20 / #21

The existing singleton socket now uses a bounded, session-scoped deduplicator for persisted `biz_notification` IDs. Missing IDs are accepted because identity cannot safely be invented. The cache is cleared on disconnect/session reset. The follow-up CI PR corrected the FIFO eviction regression test; no production duplication was introduced.

### Business notification refresh race — Frontend #22 (open at time of this document)

Research found a separate client-side race in `business_notifications_screen.dart`: initial load, realtime refresh, pull-to-refresh and pagination can overlap. A slower earlier response could overwrite a newer canonical read, and a superseded initial/pagination request could leave loading flags stuck.

The implementation adds a monotonic refresh generation. Superseded reads are discarded; stale initial/pagination requests also release their loading state; successful realtime refreshes clear the initial loading state; realtime refresh failures remain best-effort.

The change is deliberately confined to the existing notification screen. No socket, API, cache, event bus or state store was added.

## 6. Important research conclusions

### Do not duplicate the escrow terminal-state guard

The previously suspected `markSatisfied()` terminal-state race was investigated further. Existing database-level SmartEscrow transition protection already rejects terminal-state regression such as `SETTLED → PENDING_SETTLEMENT`. A second trigger/state machine would duplicate an existing invariant.

The remaining concern is request-level convergence/error semantics under concurrent settlement, not an unprotected database terminal-state overwrite.

### Do not treat every notification as an order mutation

`invoice_paid` is a separate invoice-domain event. The Business Portal therefore invalidates invoice queries directly instead of broadly invalidating order projections. Persisted `ORDER_*` notifications are different because their notification payload contains order identity and can legitimately converge the order projection.

### Keep the event transport singular

The audited implementations all extend existing singleton Socket.IO ownership. No new client event bus, second socket, financial shadow state store or duplicate API source has been introduced.

## 7. Current verification discipline

For each substantive batch:

1. inspect the producer and all affected consumers;
2. inspect existing state/query keys and lifecycle ownership;
3. identify existing invariants before adding a new one;
4. implement the smallest coherent cross-layer change;
5. compare the branch against current `main`;
6. run the repository's complete CI gate;
7. inspect the final diff for accidental duplication or unrelated formatting;
8. merge only the verified head;
9. update this planning record with implementation evidence.

## 8. Next system audit

The next major workstream is the financial event matrix across:

- escrow funded / satisfied / disputed / settled / refunded;
- invoice payment;
- withdrawal progress / settlement / failure;
- balance mutation;
- admin reconciliation exceptions;
- duplicate delivery and reconnect behavior;
- stale terminal-state event ordering.

For every event, the required evidence is:

`producer → authorization scope → room → payload identity → consumer → invalidation/read → duplicate behavior → reconnect behavior → terminal-state ordering → tests`.

The objective is not to make every client consume every event. The objective is to ensure every client that needs a projection receives a safe convergence signal and then reads authoritative state from the correct API boundary.
