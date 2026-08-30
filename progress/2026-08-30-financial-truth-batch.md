# Financial Truth & Cross-Portal Integration Batch — 2026-08-30

## System scope

AZAMAN is being treated as one system: Flutter customer app, Business Portal, Admin Portal, backend, integrations, and this planning repository. Before implementation, affected contracts are researched together; changes are grouped into coherent batches and verified at the highest available gate.

## Verified backend financial-truth foundation

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

## Verified Business/Admin authentication integration

Business Portal PR #5, Backend PR #30 and Admin Portal PR #9 are merged.

The current cross-portal authentication model is:

- Business Portal: HttpOnly refresh session + in-memory access token + synchronized Socket.IO handshake.
- Admin Portal: dedicated `azm_admin_refresh` HttpOnly session + in-memory access token + synchronized Socket.IO handshake.
- Browser clients do not trust/persist refresh credentials in localStorage.
- Security-critical Authorization/business-selection headers are authoritative.

## Backend provider-settlement foundation — VERIFIED / MERGED

Backend PR #31 was implemented after a research pass across the finance service/controller, withdrawal controller, reconciliation worker, scheduler, Prisma transaction fields, Moolre behavior, Tatum transaction behavior, and existing frontend realtime contracts.

### Defect corrected

Fiat withdrawal reservations previously created `TransactionHistory.status = COMPLETED` before external provider settlement. That made “completed” an internal reservation state rather than a provider-settled state.

### Final lifecycle foundation

`reserve → PENDING → provider settlement → COMPLETED`

or

`reserve → PENDING → provider failure → atomic reversal → FAILED`

Backend PR #31 passed the **full Backend Test Suite run #216** and was squash-merged.

## Current hardening batch — IN PROGRESS

### Backend PR #32 — provider callback as the normal settlement path

A second research pass exposed that the newly-correct `PENDING` reservation still had legacy webhook handlers which returned a successful provider callback without immediately changing the canonical `TransactionHistory` row. Reconciliation could repair it later, but that was backwards: provider settlement should be the immediate state transition and reconciliation should be recovery.

PR #32 therefore adds:

- `services/fiatSettlementService.js` as a dedicated provider→ledger boundary;
- atomic `PENDING → COMPLETED` claim on successful provider callbacks;
- persistence of provider transaction IDs into `TransactionHistory.providerRef`;
- protection against a late SUCCESS resurrecting a FAILED withdrawal;
- a normalized Moolre/MTN webhook adapter;
- `withdrawal_progress` + `withdrawal_settled` as the shared customer realtime contract;
- `admin_alert` settlement invalidation signals for the Admin control plane;
- focused service tests for success, late-success protection and failure delegation.

Moolre's current documentation confirms `txstatus` semantics of `0=Pending`, `1=Successful`, `2=Failed`, uses `externalref` as the durable business reference, and explicitly recommends keeping uncertain operations pending until final state is confirmed and making callbacks idempotent. citeturn0search0turn0search2turn0search1

**Verification:** Backend Test Suite run #218 is currently executing. PR #32 must remain open until the complete gate is green and its final diff is re-audited.

### Architectural issue still under active hardening

The legacy `Withdrawal` model still lacks a dedicated provider-attempt/reference relation, so reconciliation can still require correlation logic. This must ultimately become an explicit durable provider-attempt identity and exception model rather than relying on timestamp/amount matching.

### Admin Portal PR #10 — realtime control-plane reconciliation

Research found a separate session-lifecycle defect: Admin session restoration refreshed the access JWT but did not establish the Admin Socket.IO connection. REST could therefore be healthy while realtime was silently disabled after a browser refresh.

PR #10 adds:

- restore-time Admin Socket.IO handshake after validated session restoration;
- global `useAdminRealtime` reconciliation boundary;
- `withdrawal_settled` invalidation for withdrawal queue, stats, profit, health and payout-review queries;
- `admin_alert` invalidation for settlement/liquidity events;
- no direct financial cache mutation from socket payloads — authoritative REST refetch remains mandatory.

Admin PR #10 is open pending its build/lint/type gates.

### Flutter PR #17 — canonical withdrawal realtime transport

A research pass compared backend `withdrawal_progress` / `withdrawal_settled` events with the singleton Flutter SocketService and existing listener ownership.

Flutter PR #17 adds only the missing transport contract:

- `onWithdrawalProgress` callback registration;
- `onWithdrawalSettled` callback registration;
- listeners for both backend events;
- safe Map normalization through the existing callback helper;
- callback cleanup with the existing singleton lifecycle.

The diff was inspected after implementation and contains no duplicated socket, no feature-owned connection, and no unrelated file changes.

## Current repository state

Merged:

- Backend #29 — financial truth / PoR foundation.
- Business Portal #5 — authenticated transport + realtime token synchronization.
- Backend #30 — dedicated Admin browser sessions.
- Admin Portal #9 — Admin session/realtime integration.
- Backend #31 — provider-settlement truth and reconciliation hardening.

Open:

- Backend #32 — provider callback authoritative settlement.
- Admin Portal #10 — admin realtime settlement reconciliation.
- Flutter #17 — canonical withdrawal realtime transport contract.

## Next substantial batches after this verification gate

1. Finish and merge the three current realtime/settlement pieces only after their full quality gates pass.
2. Replace legacy withdrawal correlation with explicit provider-attempt identity and durable reconciliation exceptions.
3. Audit all Flutter escrow/order/invoice event payloads against backend emitters and enforce contract tests.
4. Audit Business Portal mutation → notification → realtime → refetch paths and add contract coverage.
5. Build the Admin reconciliation/exception queue so unresolved financial operations become actionable work rather than dashboard anomalies.
6. Extend the same `authoritative state → domain event → realtime → client reconciliation` pattern across the competition-critical commerce journeys.

## Non-negotiable architecture

`intent → authenticated API → domain authorization → authoritative transaction → durable state → domain event → realtime/notification → client reconciliation`

Backend/domain state remains authoritative. Realtime, caches, analytics and UI projections are downstream. Every financial operation must have an idempotent identity and an observable recovery path.
