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

### Final lifecycle foundation

`reserve → PENDING → provider settlement → COMPLETED`

or

`reserve → PENDING → provider failure → atomic reversal → FAILED`

Backend PR #31 passed the full backend gate and was squash-merged.

## Backend provider callback hardening — VERIFIED / MERGED

Backend PR #32 made provider callbacks the normal immediate settlement path while reconciliation remains recovery.

- Successful callbacks atomically claim only `PENDING` canonical ledger rows.
- Provider transaction identifiers are persisted into `TransactionHistory.providerRef`.
- Late SUCCESS cannot resurrect FAILED.
- Moolre/MTN transport parsing is separated from ledger mutation.
- Customer realtime uses `withdrawal_progress` + `withdrawal_settled`.
- Admin receives settlement invalidation signals.
- Complete backend coverage passed after the final fixture correction.

## Backend provider-attempt identity — VERIFIED / MERGED

Backend PR #33 passed **Azaman Test Suite #222** in full, including database schema application, Prisma generation and tests, and was squash-merged.

### Implementation

- Durable `ProviderSettlementAttempt` identity/audit table.
- Unique `(provider, providerReference)` constraint.
- Direct link to canonical `TransactionHistory`.
- Provider transaction ID, status, timestamps, failure reason and metadata.
- Idempotent provider-attempt upsert service.
- Moolre and MTN callbacks persist provider identity at the webhook boundary.
- PENDING callbacks are recorded without mutating financial state.
- Reconciliation records MTN provider identity as recovery evidence.
- Reconciliation cadence is 30 seconds.
- Regression coverage for creation, duplicate callbacks, successful settlement, late SUCCESS protection and async failure.

`ProviderSettlementAttempt` is an identity/audit layer, **not** a second ledger. `TransactionHistory` remains the financial source of truth.

## Backend reconciliation exception safety — VERIFIED / MERGED

Backend PR #34 passed its full backend gate and was squash-merged.

### Implementation

- Durable `ReconciliationException` operational queue.
- Exception upserts are idempotent while OPEN.
- Missing TransactionHistory matches become explicit exceptions.
- Multiple candidate matches become explicit exceptions and are never mutated or sent to the provider.
- Provider-status failures become durable exceptions.
- Reversal failures become durable exceptions alongside the existing Admin alert.
- Unexpected provider statuses become durable exceptions.
- Regression coverage protects missing and ambiguous correlation paths plus exception idempotency.

This changed reconciliation from a silent best-effort mechanism into an observable fail-safe process.

## Backend direct withdrawal-to-ledger identity — VERIFIED / MERGED

Backend PR #35 passed the complete backend gate after a fixture correction and was squash-merged.

### Implementation

- `Withdrawal.transactionHistoryId` provides durable canonical identity.
- Partial uniqueness and FK constraints protect the identity bridge.
- Historical backfill links only exact single-candidate matches.
- Ambiguous/missing records remain unresolved rather than being guessed.
- New mirror rows are linked only when exactly one strict candidate exists.
- Reconciliation prefers durable identity before legacy correlation.
- Regression coverage verifies linked rows do not fall back to heuristic matching.

## Backend provider callback monotonicity — IN PROGRESS / PR #37

A new research pass covered the provider-attempt service, canonical fiat settlement service, provider callback boundary, migration, existing provider-attempt tests and reconciliation architecture.

### Defect addressed

Provider callbacks can arrive out of order. The previous provider-attempt upsert could overwrite a terminal `COMPLETED` attempt with `FAILED`, or a terminal `FAILED` attempt with `COMPLETED`, even though the canonical ledger correctly refused the contradictory financial mutation.

That could leave provider evidence inconsistent with the authoritative ledger and make operational reconciliation misleading.

### Implementation in PR #37

- Provider attempt statuses are validated at the boundary.
- `COMPLETED` and `FAILED` are terminal and cannot regress.
- Late contradictory callbacks may enrich provider transaction metadata but cannot rewrite terminal status.
- The SQL `ON CONFLICT` path applies the same monotonic rule at the database boundary, protecting concurrent callbacks.
- Regression coverage covers both terminal directions.
- `TransactionHistory` remains the financial source of truth.

PR #37 is open pending the full backend quality gate and final diff/duplication audit.

## Admin Portal settlement realtime — MERGED / VERIFIED

Admin PR #10 corrected restore-time realtime establishment and introduced a global settlement reconciliation boundary. Withdrawal queue, statistics, profit, health and payout-review projections are invalidated from authoritative backend events; socket payloads never become financial truth.

## Flutter withdrawal realtime transport — MERGED / VERIFIED

Flutter PR #17 adds the canonical `withdrawal_progress` / `withdrawal_settled` transport contract to the singleton SocketService. No second socket or feature-owned connection was introduced.

## Business Portal event-contract hardening — MERGED / VERIFIED

Business Portal PR #6 passed Business Portal CI #9 and its final duplication audit, then was squash-merged.

The event layer now classifies backend `BizNotifType` events into projection invalidation domains covering orders, invoices, reservations, transit, Dine-In, KYB/trust, marketing and notifications while preserving singleton socket ownership.

## Current repository state

Merged:

- Backend #29 — financial truth / PoR foundation.
- Business Portal #5 — authenticated transport + realtime token synchronization.
- Backend #30 — dedicated Admin browser sessions.
- Admin Portal #9 — Admin session/realtime integration.
- Backend #31 — provider-settlement truth and reconciliation hardening.
- Backend #32 — provider callback authoritative settlement.
- Backend #33 — durable provider-attempt identity.
- Backend #34 — reconciliation exception safety.
- Backend #35 — durable Withdrawal → TransactionHistory identity.
- Admin Portal #10 — Admin settlement realtime reconciliation.
- Flutter #17 — canonical withdrawal realtime transport.
- Business Portal #6 — business event contract hardening.

Open:

- Backend #37 — provider-attempt terminal-state monotonicity.

## Next substantial batches

1. Finish and merge Backend #37 only after the complete quality gate and final duplication audit.
2. Expose the new Withdrawal ↔ TransactionHistory relation natively in Prisma without schema drift.
3. Complete the Admin reconciliation exception queue with claim/resolve/audit semantics and realtime updates.
4. Audit all Flutter escrow/order/invoice event payloads against backend emitters and enforce contract tests.
5. Audit Business Portal invoice/reservation/transit/Dine-In mutation → event → refetch paths against actual query keys and backend emitters.
6. Extend the authoritative-state → domain-event → realtime → client-reconciliation pattern across competition-critical commerce journeys.

## Non-negotiable architecture

`intent → authenticated API → domain authorization → authoritative transaction → durable state → domain event → realtime/notification → client reconciliation`

Backend/domain state remains authoritative. Realtime, caches, analytics and UI projections are downstream. Every financial operation must have an idempotent identity and an observable recovery path.
