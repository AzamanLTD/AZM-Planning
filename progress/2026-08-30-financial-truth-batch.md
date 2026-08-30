# Financial Truth & Cross-Portal Integration Batch — 2026-08-30

## System scope

This work treats AZAMAN as one system: Flutter customer app, Business Portal, Admin Portal, backend, integrations, and this planning repository. The implementation rule is to research the complete affected contract before changing code, then verify the complete path rather than stopping at a local compile.

## Backend financial-truth batch — merged

Backend PR #29 (`feat(finance): harden proof-of-reserves and reconciliation`) passed the complete CI gate: **67 suites / 652 tests**. It was then squash-merged.

Implemented:

- Immutable per-user Proof-of-Reserves commitments per snapshot.
- Snapshot and leaves created in one Serializable transaction.
- Historical verification from snapshot-time state.
- Read-only public PoR endpoint; explicit admin refresh mutation.
- Hourly distributed PoR worker.
- Journal + PoR integrity surface.
- Explicit exception state for unbalanced, incomplete, under-backed, or stale financial truth.
- Merkle regression coverage.
- USDT/GHS currency-boundary correction: GHS fiat liquidity cannot inflate USDT reserve backing.
- Historical PoR evidence retained with restrictive foreign keys.

### E2E cleanup correction

An intermediate cleanup implementation incorrectly guessed Prisma delegates and also risked replacing the full smoke suite. The complete original smoke suite was restored. The cleanup was then rebuilt only after researching the active Prisma schema and security controller:

- `BusinessOrderItem` is not an active Prisma delegate in the current schema and is not used by this smoke fixture.
- PIN is `User.pinHash`, not a `UserPin` model.
- Cleanup is fixture-scoped using the exact smoke-test users, business profile, and reservation rather than deleting unrelated database state.
- Cleanup errors are now re-thrown so a teardown defect cannot be hidden by a warning.

The corrected head passed the full gate before PR #29 was merged.

## Cross-portal transport audit

### Flutter customer app

`AZM-frontend/lib/config.dart` centralizes environment backend/socket URLs. `lib/services/api_client.dart` uses single-flight refresh. The realtime gap where HTTP could rotate a JWT while Socket.IO retained an expired handshake credential was corrected; refreshed credentials now cause the singleton socket to re-authenticate.

### Business Portal

Research covered `src/lib/apiCore.js`, `src/lib/AuthContext.jsx`, `src/lib/socket.js`, the backend business-session controller/routes, and backend Socket.IO authentication.

A real defect was found in the active code: `options.headers` was spread after computed security headers. This allowed a caller to replace `Authorization` or `x-admin-business-id`. The integration branch now:

- normalizes caller headers with `Headers`;
- makes Authorization authoritative;
- makes business selection authoritative;
- retains credentials for the HttpOnly session cookie;
- retains the 30-second timeout and single-flight refresh;
- re-authenticates the existing Socket.IO connection after HTTP token rotation.

Business Portal PR #5 is open and awaiting its production build gate.

### Admin Portal

Research covered `src/lib/AuthContext.jsx`, `src/lib/api.js`, the portal source tree, and the backend's existing business-session architecture.

The Admin Portal currently stores its access JWT in localStorage and sends it on every request. Its API layer already has credentials enabled and a 30-second timeout, but its 401 behavior simply discards the token instead of using the backend refresh-token lifecycle.

The next implementation is now underway in the backend branch `admin-session-bridge`:

- dedicated `azm_admin_refresh` HttpOnly cookie namespace;
- admin-only browser login bridge;
- explicit revocation if a non-admin somehow reaches the bridge;
- refresh/bootstrap role revalidation;
- logout revocation and cookie clearing;
- controller-level regression tests.

This is intentionally a separate authentication contract from Business Portal sessions rather than reusing a business cookie.

## Backend realtime contract

Socket.IO remains JWT-authenticated server-side. User, balance, order, and group rooms are authorization-bound. Client socket payloads are treated as invalidation/notification signals; authoritative state remains the API/database.

## System-level command contract

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → financial/operational event → realtime notification → client cache reconciliation`

No portal should invent state locally after a successful mutation. Events should trigger refetch/invalidation, while the API remains authoritative.

## Verification state

- Backend financial-truth PR #29: **MERGED after 67/67 suites and 652/652 tests passed**.
- Backend E2E cleanup warning: **fixed and verified by the final 67-suite run**.
- Business Portal integration PR #5: **OPEN**, awaiting CI.
- Admin browser-session backend branch: **IMPLEMENTATION IN PROGRESS**, with focused controller coverage added.
- Admin Portal transport hardening from the prior batch: **green**.

## Next substantial sequence

1. Finish and verify the dedicated Admin browser-session contract across backend + Admin Portal.
2. Add Admin API-core refresh/replay and socket re-authentication against the new session lifecycle.
3. Add cross-portal mutation/event contract coverage for Business Portal and Admin Portal.
4. Audit Flutter socket event names/payloads against backend handlers and close drift.
5. Connect external provider transaction identity to durable financial reconciliation exceptions and the Admin review workflow.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
