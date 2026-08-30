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

### Critical currency-boundary correction

A deeper schema audit found that `User.availableBalance`, `escrowLockedBalance`, `vendorUnallocatedBalance`, and `disputeEscrowBalance` are USDT liabilities, while `SystemFiatPool.balance` is a separate fiat pool (GHS). The first PoR implementation incorrectly added the fiat pool to the USDT reserve numerator.

That is now corrected. PoR reserve backing is calculated only from the USDT-reserve pools (`SystemMasterCrypto` + `SystemHotWallet`). The fiat pool remains visible separately in the snapshot breakdown for reconciliation visibility but can never inflate the USDT backing ratio.

A pure regression test now locks this boundary so a future refactor cannot silently reintroduce cross-currency arithmetic.

## CI failure root cause and correction

The first PR run reached application assertions but failed in E2E teardown because schema-guessing cleanup referenced nonexistent Prisma delegates. CI research showed the suite uses a disposable PostgreSQL database and applies the schema before tests. The test file was restored to the complete original smoke coverage and the guessed teardown was removed entirely.

The immediately following run was cancelled by GitHub when the reserve-currency correction superseded it. The current authoritative verification is backend CI run #209 on the latest head.

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

Research found the same request configuration-ordering hazard. The central API layer is now hardened so computed authentication headers cannot be overwritten by `options`, credentials are consistently included, and requests abort after 30 seconds rather than hanging indefinitely.

Admin Portal commit: `5f1d22dc2b5be89865f2e2476ce0f42771059b39`

The Admin Portal still uses standard `/api/auth/login` with a browser-accessible access token in localStorage and redirects to login when the 15-minute access token expires. The backend standard refresh endpoint is available, but moving the admin portal to an HttpOnly browser session requires a dedicated admin-session contract rather than incorrectly reusing the business portal cookie contract. That remains the next authentication batch.

### Backend realtime contract

Socket.IO is JWT-authenticated server-side and automatically joins `user_<id>` and `balance_room_<id>`. Order rooms validate that the user is either the customer or the business owner. Group chat has its own socket service and validates membership before joining. Clients therefore treat socket events as invalidation/notification signals, not as independent financial truth.

## System-level principle reinforced by this batch

Commands must follow one authoritative path:

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → financial/operational event → realtime notification → portal/app cache reconciliation`

Realtime messages are nudges, not financial truth. Clients refetch authoritative state after events rather than treating socket payloads as balances, order state, inventory state, or settlement truth.

## Verification status

Backend PR #29 remains intentionally open until the complete backend gate validates the restored full E2E smoke suite and the entire financial-truth batch together.

Current backend CI: run #209, executing against head `b2701b5cd5f260c25c7767497a9cedaf8a167058` (service change) with test commit `c56be194f600b2b4985071d11a5672d8cb9737c8` as the latest PR head.

Admin Portal transport CI run #22 is green. Business Portal transport CI run #3 is green. Flutter's quality workflow is PR-oriented; its realtime-auth fix is intended to be carried into the next coherent Flutter PR with contract coverage.

## Next substantial integration batch

1. Complete backend PR #29 gate using the restored full smoke suite and corrected currency boundary.
2. Build the dedicated Admin browser-session contract end-to-end (backend cookie bridge + admin AuthContext + API core + expiry/reconnect tests).
3. Add cross-portal contract coverage for business/admin actions that mutate backend state and then emit realtime invalidation events.
4. Audit Flutter service/socket event names and payload shapes against backend handlers and add contract tests where drift exists.
5. Connect external provider state to the financial reconciliation exception queue.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
