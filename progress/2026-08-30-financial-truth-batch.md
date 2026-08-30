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

## Backend provider-settlement truth — VERIFIED / MERGED

Backend PR #31 was implemented after a research pass across the finance service/controller, withdrawal controller, reconciliation worker, scheduler, Prisma transaction fields, Moolre behavior, Tatum transaction behavior, and existing frontend realtime contracts.

### Defect corrected

Fiat withdrawal reservations previously created `TransactionHistory.status = COMPLETED` before external provider settlement. That made “completed” an internal reservation state rather than a provider-settled state.

### Final lifecycle

`reserve → PENDING → provider settlement → COMPLETED`

or

`reserve → PENDING → provider failure → atomic reversal → FAILED`

### Implementation

- Fiat reservation creates `TransactionHistory` as `PENDING`.
- PENDING reservations can be reversed.
- Reversal claims the row atomically before refunding, preventing concurrent retry double-refunds.
- Reconciliation advances the canonical transaction to COMPLETED only after provider success.
- Existing `TransactionHistory.providerRef` stores the provider transaction/reference identifier.
- Provider failure references are retained before reversal.
- `withdrawal_settled` realtime payload carries provider transaction reference when available.
- Distributed reconciliation cadence is 30 seconds instead of the previous 5-minute recovery window.
- Focused regression tests cover successful provider settlement and asynchronous provider failure.

Backend PR #31 passed the **full Backend Test Suite run #216** and was squash-merged. The exact PR head was verified before merge.

### Remaining architectural limitation identified during the same audit

The legacy `Withdrawal` model does not have a dedicated provider-attempt/reference relation, so the reconciliation worker still locates the canonical TransactionHistory using the existing user + amount + timestamp correlation. This is a recovery mechanism, not the desired long-term identity model.

The Moolre webhook already emits immediate `withdrawal_progress` UX events, but its success branch still returns the existing transaction status rather than directly updating TransactionHistory. The 30-second reconciliation worker now repairs that canonical state. The next backend batch should make the authenticated webhook itself the immediate state transition and retain the worker strictly as recovery/reconciliation.

Tatum's current API contract returns a `txId` for successful transaction broadcasts; production code must record provider identifiers from the actual provider response rather than synthesizing fake transaction hashes. citeturn0search6turn0search7

## Flutter realtime contract — IN PROGRESS / PR #17

A research pass compared backend `withdrawal_progress` / `withdrawal_settled` events with the singleton Flutter SocketService and existing listener ownership.

Flutter PR #17 adds only the missing transport contract:

- `onWithdrawalProgress` callback registration;
- `onWithdrawalSettled` callback registration;
- listeners for both backend events;
- safe Map normalization through the existing callback helper;
- callback cleanup with the existing singleton lifecycle.

The diff was inspected after implementation and contains no duplicated socket, no feature-owned connection, and no unrelated file changes.

Flutter CI run #137 is currently in progress. Do not merge PR #17 until the full quality/build gate is green.

The next consumer layer should subscribe through this singleton and reconcile authoritative withdrawal state through the existing REST/provider path; socket payloads must not become a second financial source of truth.

## Current repository state

Merged:

- Backend #29 — financial truth / PoR foundation.
- Business Portal #5 — authenticated transport + realtime token synchronization.
- Backend #30 — dedicated Admin browser sessions.
- Admin Portal #9 — Admin session/realtime integration.
- Backend #31 — provider-settlement truth and reconciliation hardening.

Open:

- Flutter #17 — canonical withdrawal realtime transport contract.

## Next substantial batches

1. Finish Flutter #17 and wire its callbacks into the authoritative withdrawal state provider/UI after CI verification.
2. Make the Moolre authenticated webhook update TransactionHistory immediately on SUCCESS/FAILED; keep reconciliation as recovery.
3. Replace the legacy withdrawal correlation heuristic with explicit provider-attempt identity and durable reconciliation exceptions.
4. Audit all Flutter escrow/order/invoice event payloads against backend emitters and enforce contract tests.
5. Audit Business Portal mutation → notification → realtime → refetch paths and add contract coverage.
6. Build the Admin reconciliation/exception queue so unresolved financial operations become actionable work rather than dashboard anomalies.
7. Continue the competition sequence from the master roadmap: trustworthy core first, then polished retail + signature escrow workspace, then vertical breadth.

## Non-negotiable architecture

`intent → authenticated API → domain authorization → authoritative transaction → durable state → domain event → realtime/notification → client reconciliation`

Backend/domain state remains authoritative. Realtime, caches, analytics and UI projections are downstream. Every financial operation must have an idempotent identity and an observable recovery path.
