# Financial Truth Batch — 2026-08-30

## Batch scope

This is a coherent P0 financial-truth hardening batch plus the first cross-portal integration audit. The work is intentionally being treated as one system: Flutter customer app, Business Portal, Admin Portal, backend, and AZM-Planning.

## Backend PR #29 — implemented

- Immutable per-user Proof-of-Reserves commitments are persisted per snapshot.
- Snapshot + all leaves are created in one Serializable transaction.
- Historical user verification uses the balance state committed at snapshot time.
- Public Proof-of-Reserves GET is read-only and cannot create unbounded snapshots.
- Snapshot creation is an explicit admin mutation (`POST /api/proof-of-reserves/refresh`).
- Hourly Proof-of-Reserves snapshot worker registration has been added to the distributed scheduler.
- Admin journal integrity endpoint combines double-entry trial balance and PoR commitment coverage.
- Integrity is exceptional when the journal is unbalanced, the snapshot is incomplete, reserves are under-backed, or the latest snapshot is older than two hours.
- Public PoR returns an explicit unavailable response when no snapshot exists rather than fabricating a successful state.
- Merkle regression tests cover deterministic roots, odd leaves, multiple positions, tampering, and index binding.

## CI failure root cause and correction

The first PR run reached the application assertions successfully but failed in E2E teardown because the test cleanup guessed Prisma delegates that are not present in the authoritative schema. Research of the CI workflow showed the suite runs against a disposable PostgreSQL database and applies the schema before testing. The test file was therefore restored to the complete original smoke coverage and the schema-guessing teardown was removed entirely.

The subsequent run after restoration is now the authoritative verification run; the earlier green run is not treated as proof because it had accidentally reduced the smoke suite.

## Cross-portal integration audit

### Flutter customer app

`AZM-frontend/lib/config.dart` centralizes environment-specific backend URLs and socket URL derivation. `lib/services/api_client.dart` already performs refresh-token rotation with a single in-flight refresh promise and normal request timeouts.

A realtime consistency gap was identified: HTTP could silently rotate the access JWT while the singleton Socket.IO connection continued using the expired JWT. After successful refresh, the client now forces the singleton Socket.IO service to reconnect using the fresh token. Reconnect failure is non-fatal to the HTTP refresh path.

Flutter commit: `0bae665057a52d54ae62fe2464b058569068bacf`

### Business Portal

The portal uses the backend's dedicated browser-session flow with an HttpOnly refresh cookie and an in-memory access token. Socket.IO uses the same access token in the handshake, while business notification events invalidate React Query caches so the server remains authoritative.

A real request-layer defect was identified: `src/lib/apiCore.js` spread `...options` over the computed fetch configuration, allowing a caller-supplied `options.headers` to replace Authorization and business-selection headers. This is fixed, and a 30-second abort timeout now prevents indefinitely hanging portal actions.

Business Portal commit: `ca75a6c4b9a825fd29eae1201e7214f2c692f75d`

### Admin Portal

Research of the admin request layer found the same configuration-ordering hazard. The central API layer has now been hardened so computed authentication headers cannot be overwritten by the options object, requests include credentials consistently, and hung requests abort after 30 seconds.

Admin Portal commit: `5f1d22dc2b5be89865f2e2476ce0f42771059b39`

The Admin Portal still uses standard `/api/auth/login` with a browser-accessible access token in localStorage and redirects to login when the 15-minute access token expires. The backend standard refresh endpoint is available, but moving the admin portal to an HttpOnly browser session requires a separate admin-session contract rather than incorrectly reusing the business portal cookie contract. That remains a deliberate next batch, not a shortcut.

### Backend realtime contract

Socket.IO is JWT-authenticated server-side and automatically joins `user_<id>` and `balance_room_<id>`. Order rooms validate that the user is either the customer or the business owner. Group chat has its own socket service and validates membership before joining. Clients therefore treat socket events as invalidation/notification signals, not as independent financial truth.

## System-level principle reinforced by this batch

Commands must follow one authoritative path:

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → financial/operational event → realtime notification → portal/app cache reconciliation`

Realtime messages are nudges, not financial truth. Clients refetch authoritative state after events rather than treating socket payloads as balances, order state, inventory state, or settlement truth.

## Verification status

Backend PR #29 remains intentionally open until the complete backend gate validates the restored full E2E smoke suite and the entire financial-truth batch together.

Backend CI run #208 is executing that restored full suite now.

Admin and Business Portal transport commits have their repository CI workflows configured for main pushes and pull requests. Flutter's quality workflow is PR-oriented; its next coherent Flutter PR should include this realtime-auth fix plus the related contract tests rather than creating one-change verification noise.

## Next substantial integration batch

1. Complete backend PR #29 gate using the restored full smoke suite.
2. Build the dedicated Admin browser-session contract end-to-end (backend cookie bridge + admin AuthContext + API core + expiry/reconnect tests).
3. Add cross-portal contract coverage for business/admin actions that mutate backend state and then emit realtime invalidation events.
4. Audit Flutter service/socket event names and payload shapes against backend handlers and add contract tests where drift exists.
5. Connect external provider state to the financial reconciliation exception queue.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
