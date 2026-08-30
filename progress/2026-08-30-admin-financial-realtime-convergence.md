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

Admin Portal PR #14 modifies only `src/hooks/useAdminRealtime.js`.

It adds convergence handling for:

- `escrow_funded`
- `escrow_settled`
- `escrow_pending_settlement`
- `escrow_disputed`
- `escrow_resolved`
- `escrow_terms_updated`
- `invoice_paid`
- `order:location`
- `order:status`
- `order:eta`
- legacy order event aliases
- `business_order_delivered`
- `balance_update`

Escrow events invalidate the existing escrow/dispute/admin financial projections. Order events invalidate only admin operational projections. Balance and withdrawal events invalidate existing financial projections.

Socket payloads are deliberately not copied into financial React Query state. Events remain convergence signals and canonical API responses remain authoritative.

## Duplication audit

No second socket, event bus, financial cache, or state store was created. Existing `adminSocket.js` remains the sole transport and existing `useAdminData.js` query keys remain the sole data surfaces.

## Validation state

PR #14 is open while its CI run executes. It is intentionally not recorded as merged until the exact head SHA has a successful CI result.

## Next audit

After Admin CI is green, continue the event matrix through Business Portal and Flutter for terminal escrow/refund/settlement events, duplicate delivery, reconnects and stale-event ordering. Avoid adding new event names unless source tracing proves the current contract cannot express the required state transition.
