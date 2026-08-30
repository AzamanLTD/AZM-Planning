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

The first PR run reached the application assertions successfully but failed in E2E teardown because the test cleanup guessed Prisma delegates that are not present in the authoritative schema (`reservation` / `userPin` were referenced as delegates in teardown code that did not match the generated client).

Research of the CI workflow showed that the E2E suite runs against a disposable PostgreSQL database created for the workflow. The workflow explicitly applies the schema to that disposable database before the test suite. The correct fix is therefore **not** to add more guessed model cleanup or raw-table deletion. The smoke test now leaves its uniquely timestamped fixtures in the disposable CI database and does not perform schema-guessing teardown.

Commit: `512a3a9749d5a52b277fa400aa6c4cce6f51ecea`

## Cross-portal integration audit

### Flutter customer app

`AZM-frontend/lib/config.dart` currently centralizes environment-specific backend URLs and socket URL derivation. Production, staging, and development are explicit, which is the correct foundation for eliminating environment drift.

### Business Portal

The portal uses the backend's dedicated browser-session flow:

- `POST /api/auth/business-session/login`
- `POST /api/auth/business-session`
- `POST /api/auth/business-session/logout`

Refresh credentials remain in an HttpOnly cookie and access tokens remain in memory. Socket.IO uses the same access token in the handshake and the portal refreshes React Query caches from backend business-notification events.

A real request-layer defect was identified: `src/lib/apiCore.js` constructed authentication headers and then spread `...options` over the whole fetch configuration. A caller supplying `options.headers` could therefore replace the computed Authorization/business-selection headers. This is now fixed, and a 30-second abort timeout was added so a hung backend request cannot leave a portal action indefinitely pending.

Business Portal commit: `ca75a6c4b9a825fd29eae1201e7214f2c692f75d`

### Admin Portal

The Admin Portal currently uses the standard `/api/auth/login` token flow and stores its access JWT in localStorage. Its central request layer has the same structural `...options` ordering hazard as the Business Portal layer and currently forces a login redirect on token expiry rather than using the backend refresh-token rotation flow. This is a next integration-hardening target, but it should be implemented only after inspecting the admin auth/session contract and existing backend refresh tests together.

### Backend

The backend auth contract explicitly exposes both standard refresh-token rotation and the dedicated business browser-session flow. Socket.IO verifies JWTs server-side. The business portal's socket model is therefore aligned with the backend's intended trust boundary.

## System-level principle reinforced by this batch

Commands must follow one authoritative path:

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → financial/operational event → realtime notification → portal/app cache reconciliation`

Realtime messages are nudges, not financial truth. Clients refetch authoritative state after events rather than treating socket payloads as balances, order state, inventory state, or settlement truth.

## Verification status

Backend PR #29 remains intentionally open until the complete backend gate validates migration, generated Prisma client, service, routes, worker registration, E2E suite, and regression tests together.

The latest correction has triggered backend CI run #207.

## Next substantial integration batch

1. Complete backend PR #29 gate.
2. Harden Admin Portal request/session transport after researching standard refresh-token semantics and existing admin auth tests.
3. Add cross-portal contract coverage for business/admin actions that mutate backend state and then emit realtime invalidation events.
4. Audit Flutter service/socket layers against the same endpoint and event contracts.
5. Connect external provider state to the financial reconciliation exception queue.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
