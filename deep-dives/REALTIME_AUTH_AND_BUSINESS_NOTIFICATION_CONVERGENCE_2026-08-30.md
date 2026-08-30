# Realtime Auth + Business Notification Convergence — 2026-08-30

## Status

**VERIFIED / MERGED**

## Scope

This batch closed two cross-system realtime lifecycle gaps found by tracing the existing implementations before changing them:

1. Business Portal could receive a fresh REST access token while its already-connected Socket.IO singleton continued using the previous JWT; logout revoked REST state but did not tear down the privileged `admin_spy_room` socket.
2. Escrow/business notifications were persisted into the canonical `BusinessNotification` feed, but that persistence chokepoint did not emit the existing `biz_notification` Socket.IO convergence signal. The Business Portal therefore could not react immediately to persisted escrow notifications, and its existing `biz_notification` handler only invalidated notification queries rather than the affected order projection.

## Implemented changes

### Admin Portal

Merged **PR #13** (`2c9e68d625acd7af0d536859665cd24cc3470c77`).

Affected files:

- `src/lib/adminSocket.js`
- `src/lib/AuthContext.jsx`

Behavior now:

- session restoration/login rotates the socket JWT when a fresh access token differs;
- auth failure disconnects the privileged socket;
- logout disconnects the privileged socket;
- the existing singleton transport and `join_admin_spy` handshake remain unchanged.

CI run 32 passed on the exact PR head before merge.

### Backend

Merged **PR #43** (`9dca0c88b638280203759699c0aff296d1100728`).

Affected files:

- `services/bizNotificationService.js`
- `src/sockets/socketServices.js`
- `__tests__/business-notifications.unit.test.js`

Behavior now:

- the existing Socket.IO server instance is wired once into the BusinessNotification service at socket-service bootstrap;
- successful `BusinessNotification` persistence emits exactly one `biz_notification` signal to the existing authenticated `user_<userId>` room;
- the payload contains only convergence/correlation data needed by the client (`notificationId`, `businessProfileId`, `type`, order identifiers, escrow identifier and creation timestamp);
- socket failure is best-effort and cannot turn notification persistence into a financial/request failure;
- regression tests cover successful emission and transport-unavailable persistence.

Backend Test Suite run 250 passed on the exact PR head before merge.

### Business Portal

Merged **PR #11** (`27617f01e44ad6ed28463e52d47ecf3b18484e4c`).

Affected file:

- `src/lib/realtimeQueryBridge.js`

Behavior now:

- `biz_notification` continues to invalidate notification queries;
- persisted `ORDER_*` business notifications also invalidate the affected order detail/list/stat queries using the payload's `orderId`;
- unrelated notification types do not trigger broad order invalidation;
- no second realtime channel or client-side source of truth is introduced.

Business Portal CI run 19 passed on the exact PR head before merge.

## Research conclusions

### Do not duplicate financial state guards

The existing escrow database invariants already prevent terminal escrow states from regressing. The realtime layer must not become a second financial state machine.

### Socket events are convergence signals

The backend/database/API remains authoritative. A client receiving `biz_notification` invalidates/refetches canonical resources rather than treating the event payload as authoritative financial state.

### Existing room contract is sufficient

The backend connection handler already authenticates sockets and automatically joins `user_<userId>`. Business owner identity is resolved from the linked BusinessOrder's BusinessProfile. The batch reuses that room rather than creating a business-notification room.

### Redis scaling remains compatible

`server.js` already installs the optional Socket.IO Redis adapter before socket services are instantiated. Wiring the same `io` instance into BusinessNotification therefore participates in the existing horizontal fan-out mechanism; no second adapter or event bus is required.

## Cross-system flow after this batch

```text
Escrow financial mutation
        |
        +--> authoritative DB state / ledger
        |
        +--> BusinessNotification persisted
                    |
                    +--> biz_notification
                              |
                    +---------+---------+
                    |                   |
              Business Portal      notification feed
                    |
             invalidate order queries
                    |
             canonical API refetch
```

Admin authentication follows the equivalent lifecycle rule:

```text
REST session rotation/logout
        |
        +--> access-token lifecycle
        |
        +--> Socket.IO singleton lifecycle
                    |
             connect / reconnect
                    |
             authenticated admin_spy room
```

## Verification

At the time of this record:

- Backend PR #43: merged after full Test Suite passed.
- Business Portal PR #11: merged after CI passed.
- Admin Portal PR #13: merged after CI passed.
- No speculative financial-state rewrite was introduced.
- The implementation diffs were compared against their direct `main` bases and checked for duplicated transport/state machinery.

## Remaining work

The next financial-event audit should trace customer-facing escrow events through Flutter with the same discipline. In particular, verify whether Flutter receives each persisted escrow lifecycle transition through an existing event or notification surface, and ensure reconnect/missed-event behavior converges through canonical API state rather than adding another transport.
