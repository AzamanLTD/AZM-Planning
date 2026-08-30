# Financial Truth & Cross-Portal Integration Batch — 2026-08-30

## System scope

AZAMAN is being treated as one system: Flutter customer app, Business Portal, Admin Portal, backend, integrations, and this planning repository. Before implementation, affected contracts are researched together; changes are grouped into coherent batches and verified at the highest available gate.

## Completed backend financial-truth batch

Backend PR #29 was squash-merged after the full **67-suite / 652-test** gate passed.

Key results:

- Immutable per-user Proof-of-Reserves commitments per snapshot.
- Historical verification against snapshot-time balances.
- Read-only public PoR endpoint + explicit admin refresh.
- Hourly distributed PoR worker.
- Journal/PoR integrity exception surface.
- USDT/GHS reserve-boundary correction.
- Restrictive retention for historical PoR evidence.
- Complete E2E smoke suite restored after an intermediate bad cleanup implementation.
- Fixture-scoped E2E cleanup now uses only delegates verified against the active schema; `User.pinHash` is reset directly because PIN is not a separate model.
- Final corrective run completed successfully.

## Business Portal integration batch — merged

Business Portal PR #5 was merged after CI passed.

Research covered the portal API core, AuthContext, Socket.IO singleton, backend business-session controller/routes, and backend Socket.IO authentication.

Implemented:

- caller headers normalized with `Headers`;
- Authorization and business-selection headers authoritative;
- HttpOnly session credentials included;
- 30-second request timeout retained;
- single-flight refresh retained;
- Socket.IO handshake credential replaced/reconnected after HTTP token rotation.

## Admin browser-session migration — merged

Backend PR #30 and Admin Portal PR #9 are both now merged.

### Backend session contract

Dedicated `azm_admin_refresh` HttpOnly cookie namespace:

- canonical auth verification;
- ADMIN-only session boundary;
- immediate revocation of a refresh credential issued to a rejected non-admin login;
- refresh rotation through Phase-K token service;
- ADMIN revalidation after rotation;
- invalid/expired cookie clearing;
- server-side logout revocation;
- no refresh credential returned to JavaScript.

Focused tests cover cookie secrecy, role enforcement, rotation, and role downgrade handling. Backend PR #30 passed the full backend gate before merge.

### Admin Portal client

The portal no longer trusts or persists `admin_token` in localStorage.

The central API lifecycle now uses:

`admin-session/login → in-memory access token → HttpOnly refresh cookie → single-flight refresh → socket re-auth`

The Admin Socket.IO connection is centralized. AlertBanner no longer opens an independent socket using a stale localStorage token. Access-token rotation re-handshakes the existing socket.

The portal CI passed lint and build before merge.

An existing `spring.toast` runtime reference in AlertBanner was also corrected to use the actual Forge motion contract, and the complete API default export was preserved, including storefront operations.

## Flutter customer app

The customer app already has a centralized singleton SocketService and a single-flight refresh path. The current refresh flow persists the new access token and explicitly forces socket reconnection so REST and realtime use the same token generation. Listener ownership has been hardened so feature providers do not dispose the shared global socket.

The next Flutter work is a contract audit of event names/payloads against backend emitters, especially marketplace orders, escrow, invoice payment, notifications, and order tracking.

## System command contract

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → financial/operational event → realtime notification → client cache reconciliation`

Realtime payloads are invalidation/notification signals, not financial truth. Clients refetch authoritative state after mutations/events.

## Current repository state

The previously active Business Portal and Admin Portal integration PRs are merged. The dedicated Admin backend session PR is also merged. No authentication migration is considered complete merely because a browser build succeeds; the backend session contract and client contract must agree on cookie namespace, token lifecycle, role boundary, and realtime re-authentication.

## Next substantial implementation sequence

1. Verify post-merge production workflows for the backend and Admin Portal.
2. Add cross-portal contract tests for mutation → persistence → realtime event → cache reconciliation.
3. Audit Flutter socket event names/payloads against every relevant backend emitter/handler and close drift.
4. Audit Business Portal mutation/event pairs using the same contract matrix.
5. Implement durable external-provider transaction identity + reconciliation exceptions + Admin review/repair workflow.
6. Feed the resulting financial exceptions into the Admin control plane without turning dashboards into the source of truth.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
