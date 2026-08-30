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

## Backend direct withdrawal-to-ledger identity — IN PROGRESS / PR #35

The next research pass covered the current `Withdrawal` schema, `TransactionHistory`, the fiat withdrawal controller, reconciliation worker, provider-attempt identity, exception queue and worker tests.

### Defect being eliminated

`Withdrawal` historically had no direct canonical transaction reference. New and legacy reconciliation therefore depended on user + amount + five-second timestamp correlation.

### Implementation in PR #35

- Adds nullable `Withdrawal.transactionHistoryId` at the database layer.
- Adds a partial unique index and FK to `TransactionHistory.id`.
- Safely backfills only exact single-candidate historical matches.
- Leaves ambiguous/missing rows unlinked so the reconciliation exception queue can handle them.
- Adds a database trigger for newly inserted mirror rows, linking only when exactly one strict candidate exists.
- Reconciliation prefers the durable identity before using the legacy migration fallback.
- Safely promotes uniquely-correlated legacy rows into the durable link.
- Regression tests verify linked rows never fall back to heuristic matching.

### Important implementation boundary

The checked-in Prisma client currently predates the additive database column. PR #35 deliberately reads/writes the new identity bridge through SQL while preserving the existing Prisma client contract. A subsequent schema-generation cleanup should expose the relation natively once the full schema file can be regenerated and verified as part of the normal migration pipeline.

PR #35 is open and must not merge until the complete backend gate passes and the migration/worker diff is re-audited.

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
- Admin Portal #10 — Admin settlement realtime reconciliation.
- Flutter #17 — canonical withdrawal realtime transport.
- Business Portal #6 — business event contract hardening.

Open:

- Backend #35 — durable Withdrawal → TransactionHistory identity.

## Next substantial batches

1. Finish and merge Backend #35 only after the complete quality gate and final duplication/migration audit.
2. Regenerate/expose the new Withdrawal ↔ TransactionHistory relation natively in Prisma without reintroducing schema drift.
3. Expose `ReconciliationException` through the Admin Portal as an actionable war-room queue with claim/resolve/audit semantics.
4. Audit all Flutter escrow/order/invoice event payloads against backend emitters and enforce contract tests.
5. Audit Business Portal invoice/reservation/transit/Dine-In mutation → event → refetch paths against actual query keys and backend emitters.
6. Extend the same authoritative-state → domain-event → realtime → client-reconciliation pattern across competition-critical commerce journeys.

## Non-negotiable architecture

`intent → authenticated API → domain authorization → authoritative transaction → durable state → domain event → realtime/notification → client reconciliation`

Backend/domain state remains authoritative. Realtime, caches, analytics and UI projections are downstream. Every financial operation must have an idempotent identity and an observable recovery path.
