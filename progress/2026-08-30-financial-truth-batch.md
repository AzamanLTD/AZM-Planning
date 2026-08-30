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

### Admin Portal client

The portal no longer trusts or persists `admin_token` in localStorage. The central lifecycle is:

`admin-session/login → in-memory access token → HttpOnly refresh cookie → single-flight refresh → socket re-auth`

The Admin Socket.IO connection is centralized. AlertBanner no longer opens an independent stale-token socket. The existing `spring.toast` runtime reference was corrected to the Forge motion contract and the complete API export surface was preserved.

## Provider-settlement truth batch — Backend PR #31 OPEN

A research pass across `finance.service.js`, `finance.controller.js`, `withdrawalController.js`, `withdrawalReconciliationWorker.js`, `src/workers/index.js`, the Prisma withdrawal/transaction fields, Moolre disbursement behavior, and Tatum's current transaction API uncovered an important lifecycle defect.

### Defect

Fiat withdrawal reservations were writing `TransactionHistory.status = COMPLETED` before the external payout provider had settled the payout. Consequently, a customer status read could say “completed” while the provider was still processing the transfer.

The reconciliation worker also assumed the canonical transaction was already complete and only changed the mirror `Withdrawal` row.

### PR #31 changes

The new lifecycle is:

`reserve → PENDING → provider settlement → COMPLETED`

or

`reserve → PENDING → provider failure → atomic reversal → FAILED`

Implemented:

- fiat reservation writes `TransactionHistory` as `PENDING`;
- `reverseFiatWithdrawal` accepts PENDING reservations;
- reversal first claims the transaction row with a conditional status update, preventing concurrent provider retries from double-refunding;
- reconciliation changes the canonical TransactionHistory row to COMPLETED only after provider success;
- provider references are persisted using the existing `TransactionHistory.providerRef` field;
- provider failure references are retained before reversal;
- realtime `withdrawal_settled` includes the provider transaction reference when available;
- reconciliation cadence is 30 seconds through the existing distributed BullMQ scheduler;
- focused regression tests cover both provider success and asynchronous failure.

Tatum's current documentation confirms that successful Polygon transaction broadcasts return a `txId`, so provider identifiers must be recorded from the provider response rather than generated as fake transaction hashes. citeturn0search0turn0search6

### Verification status

Backend PR #31 is open. Its full backend GitHub Actions gate is currently running (run #216). It must remain open until the complete suite is green.

### Remaining provider-correlation work

The current legacy `Withdrawal` model has no dedicated reference/provider-id column; reconciliation therefore still discovers the canonical transaction through the existing user + amount + timestamp correlation and stores the provider reference on `TransactionHistory`. A future schema batch should add an explicit unique withdrawal reference/provider-attempt relation rather than extending this heuristic indefinitely.

The Moolre webhook path should also be upgraded to transition the canonical TransactionHistory row immediately on authenticated SUCCESS/FAILED callbacks; the 30-second reconciliation loop is the safety net, not the ideal realtime settlement path.

## Flutter customer app

The customer app has a centralized singleton SocketService and a single-flight refresh path. The refresh flow persists the new access token and forces socket reconnection so REST and realtime use the same token generation. Listener ownership has been hardened so feature providers do not dispose the shared global socket. The withdrawal flow already has a backend status endpoint and realtime-oriented progress contract; the next implementation step is to wire the canonical settlement events into that shared socket lifecycle instead of allowing feature-specific polling to become the source of truth.

## Business Portal realtime contract

The Business Portal uses a single Socket.IO connection and refresh-token synchronization. Backend business events include persistent BusinessNotification records plus realtime `biz_notification` nudges. The next audit should enumerate every `BusinessOrder`/invoice/escrow/KYB event, compare backend emitters against Business Portal consumers, and add contract tests for payload shape and authorization.

## System command contract

`Portal/App intent → authenticated API → domain authorization → authoritative DB mutation → provider/domain settlement → durable transaction state → realtime invalidation/event → client cache reconciliation`

Realtime payloads are invalidation/notification signals, not financial truth. Clients refetch authoritative state after mutations/events.

## Current repository state

Merged:

- Backend PR #29 — financial truth / PoR batch.
- Business Portal PR #5 — authenticated transport + realtime token synchronization.
- Backend PR #30 — dedicated Admin browser session.
- Admin Portal PR #9 — matching Admin session/realtime migration.

Open:

- Backend PR #31 — provider-settlement truth and reconciliation hardening.

## Next substantial implementation sequence

1. Finish and verify Backend PR #31; do not merge on partial/focused tests alone.
2. Upgrade Moolre webhook settlement to update canonical TransactionHistory immediately, with the 30-second worker as the recovery path.
3. Add explicit provider correlation to Withdrawal/transaction lifecycle so reconciliation no longer depends on amount/time heuristics.
4. Audit Flutter socket event names and payloads against every relevant backend emitter and wire withdrawal settlement into the shared socket state.
5. Audit Business Portal mutation/event pairs and add cross-portal contract tests.
6. Implement durable external-provider transaction identity + reconciliation exceptions + Admin review/repair workflow.
7. Surface unresolved financial exceptions in the Admin control plane without making dashboards the source of truth.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
