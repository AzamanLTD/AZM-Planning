# Admin financial realtime convergence — 2026-08-30

## Scope

This batch extends the existing Admin Portal realtime reconciliation boundary so platform-control-plane projections converge after financial, escrow, order and balance events.

## Research completed before implementation

Affected code was traced before editing:

- `AZM-adminPortal/src/lib/AuthContext.jsx` establishes and tears down the authenticated admin socket.
- `AZM-adminPortal/src/lib/adminSocket.js` owns the singleton Socket.IO connection and joins `admin_spy_room` after connection.
- `AZM-adminPortal/src/hooks/useAdminRealtime.js` is the existing React Query convergence boundary.
- `AZM-adminPortal/src/lib/useAdminData.js` defines the concrete query keys for withdrawals, disputes, dispute resolutions, stats, profit, health, audit log and escrow disputes.
- `AZM-adminPortal/package.json` already contains `socket.io-client`; no dependency addition was required.
- `AZM-backend/src/sockets/connectionHandler.js` confirms `join_admin_spy` is restricted to the ADMIN role and joins `admin_spy_room`.

## Implementation

Admin Portal PR #14 modified only `src/hooks/useAdminRealtime.js`.

It adds convergence handling for escrow, invoice, order, balance and reconciliation events while retaining the existing admin socket and React Query architecture.

## Duplication audit

No second socket, event bus, financial cache, or state store was created. Existing `adminSocket.js` remains the sole transport and existing `useAdminData.js` query keys remain the sole data surfaces.

## Validation

PR #14 CI completed successfully on implementation head `0c480af9455e0c145c00b51f4d2bc86bfd00b8c5`.

PR #14 was squash-merged into `main` as `fe35146ad502cfe1a945fa3cc69e7d6a191b714c`.

## Next audit

Continue the event matrix through Business Portal and Flutter for terminal escrow/refund/settlement events, duplicate delivery, reconnects and stale-event ordering. Avoid adding new event names unless source tracing proves the current contract cannot express the required state transition.
