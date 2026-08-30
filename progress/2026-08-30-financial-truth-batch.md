# Financial Truth & Cross-Portal Integration Batch — 2026-08-30

## System scope

AZAMAN is being treated as one system: Flutter customer app, Business Portal, Admin Portal, backend, integrations, and this planning repository. Before implementation, affected contracts are researched together; changes are grouped into coherent batches and verified at the highest available gate.

## Completed backend financial-truth batch

Backend PR #29 was squash-merged after the full **67-suite / 652-test** gate passed.

Key results:

- Immutable per-user PoR commitments per snapshot.
- Historical verification against snapshot-time balances.
- Read-only public PoR endpoint + explicit admin refresh.
- Hourly distributed PoR worker.
- Journal/PoR integrity exception surface.
- USDT/GHS reserve-boundary correction.
- Restrictive retention for historical PoR evidence.
- Complete E2E smoke suite restored after an intermediate bad cleanup implementation.
- Fixture-scoped E2E cleanup now uses only delegates verified against the active schema; `User.pinHash` is reset directly because PIN is not a separate model.
- Final corrective run completed successfully.

## Business Portal integration batch

PR #5 was merged after its CI passed.

Research covered:

- `src/lib/apiCore.js`
- `src/lib/AuthContext.jsx`
- `src/lib/socket.js`
- backend business-session controller/routes
- backend Socket.IO authentication

Implemented:

- caller headers are normalized with `Headers`;
- `Authorization` and business-selection headers are authoritative;
- HttpOnly session credentials remain included;
- 30-second request boundary retained;
- single-flight refresh retained;
- Socket.IO receives the fresh JWT after HTTP refresh and performs a new handshake.

This closes the class of failure where REST silently recovers while realtime remains authenticated with an expired JWT.

## Admin browser-session migration

### Backend PR #30 — open, verification in progress

Dedicated `azm_admin_refresh` HttpOnly session contract implemented.

The backend bridge:

- uses the canonical auth controller for credential verification;
- refuses non-ADMIN users at the session boundary;
- revokes a refresh token created for a rejected non-admin login;
- rotates refresh tokens through the existing Phase-K service;
- rechecks ADMIN role after rotation;
- clears invalid/expired cookies;
- revokes the server token on logout;
- never returns refresh credentials to browser JavaScript.

Focused controller tests cover cookie secrecy, non-admin rejection, rotation, and role downgrade handling.

Current backend verification: run #214 is executing the full backend suite.

### Admin Portal PR #9 — open, CI passed

Research covered:

- `src/lib/AuthContext.jsx`
- `src/lib/api.js`
- `src/components/admin/AlertBanner.jsx`
- `src/components/forge/ForgeLayout.jsx` / shell wiring
- backend Phase-K token lifecycle
- backend business-session architecture

Implemented:

- no admin access JWT in localStorage;
- login via `/api/auth/admin-session/login`;
- startup restore via `/api/auth/admin-session`;
- in-memory access JWT with single-flight 401 refresh;
- logout through server-side session revocation;
- authoritative security headers;
- centralized Admin Socket.IO session;
- socket re-authentication after access-token rotation;
- AlertBanner now consumes the central socket instead of opening a second stale-token connection;
- corrected the existing `spring.toast` runtime reference to the Forge motion contract;
- preserved the complete API surface, including storefront exports.

Admin Portal CI run #23: **green** (lint + build).

### Important dependency order

Admin Portal PR #9 is intentionally coupled to backend PR #30. Backend #30 must land first so the production endpoint exists before the portal switches from `/api/auth/login` to `/api/auth/admin-session/login`.

## Flutter customer app

The customer app already has a centralized singleton SocketService and a single-flight refresh path. The current refresh flow persists the new access token and explicitly forces socket reconnection so REST and realtime use the same token generation. Listener ownership has also been hardened so feature providers do not dispose the shared global socket.

The next Flutter work is a contract audit of event names/payloads against backend emitters, especially marketplace orders, escrow, invoice payment, notifications, and order tracking.

## System command contract

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → financial/operational event → realtime notification → client cache reconciliation`

Realtime payloads are invalidation/notification signals, not financial truth. Clients refetch authoritative state after mutations/events.

## Current PR state

Open PRs:

- Backend #30 — dedicated Admin session backend, full suite running.
- Admin Portal #9 — matching Admin session client, CI green.

Business Portal #5 and Backend #29 are merged.

## Next substantial implementation sequence

1. Complete backend #30's full gate and merge it if green.
2. Merge Admin Portal #9 against the landed backend contract.
3. Build cross-portal mutation/event contract tests rather than relying only on build checks.
4. Audit Flutter socket event names and payloads against every relevant backend emitter/handler and close drift.
5. Implement durable external-provider transaction identity + reconciliation exceptions + Admin review/repair workflow.

The project should not claim “integrated” merely because builds pass; an integration is complete only when authentication, command execution, persistence, event emission, and client reconciliation agree on the same contract.
