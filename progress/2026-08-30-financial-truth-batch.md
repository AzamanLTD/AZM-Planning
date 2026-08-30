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

Backend PR #31 passed the full backend gate and was squash-merged.

## Backend provider callback hardening — VERIFIED / MERGED

Backend PR #32 completed the next financial-truth step: provider callbacks are now the normal immediate settlement path while reconciliation remains recovery.

### Implementation

- `services/fiatSettlementService.js` is the provider→ledger transition boundary.
- Successful callbacks atomically claim only `PENDING` canonical ledger rows.
- Provider transaction identifiers are persisted into `TransactionHistory.providerRef`.
- Late SUCCESS cannot resurrect a FAILED withdrawal.
- Moolre/MTN transport parsing is separated from ledger mutation.
- Customer realtime uses `withdrawal_progress` + `withdrawal_settled`.
- Admin receives settlement invalidation signals.
- Focused regression coverage protects success, late-success and failure paths.

The complete Backend Test Suite was green after the final test-fixture correction, and PR #32 was squash-merged only after that gate.

## Backend provider-attempt identity — IN PROGRESS / PR #33

A second research pass covered the live `TransactionHistory` schema, finance reservation/reversal lifecycle, Moolre/MTN webhook normalization/authentication, MTN's external X-Reference-Id idempotency contract, reconciliation worker, existing settlement tests and Prisma migration conventions.

### Architectural defect being removed

The canonical ledger has `txHash`/`providerRef`, but there was no durable record representing the external provider attempt itself. That forced the recovery layer to treat provider identity as an attribute discovered after the fact.

### Implementation in Backend PR #33

- Adds `ProviderSettlementAttempt` as a durable external-identity/audit table.
- Enforces uniqueness on `(provider, providerReference)`.
- Links each attempt directly to canonical `TransactionHistory`.
- Stores provider transaction ID, status, first/last seen timestamps, terminal timestamp, failure reason and provider metadata.
- Adds an idempotent upsert service using parameterized SQL against the additive table.
- Moolre and MTN callbacks now persist provider identity at the webhook boundary.
- PENDING callbacks are recorded without mutating financial ledger state.
- Reconciliation records MTN provider identity as recovery evidence.
- Reconciliation cadence is now 30 seconds.
- Regression coverage covers creation, duplicate callback update, successful settlement, late SUCCESS protection and asynchronous failure.

### Deliberate boundary

`ProviderSettlementAttempt` is **not** a second ledger. `TransactionHistory` remains authoritative for money state.

The remaining legacy weakness is the `Withdrawal` mirror's lack of a direct canonical TransactionHistory reference. PR #33 makes provider identity durable first; the next batch should replace the remaining user+amount+timestamp recovery correlation with a direct reference and durable reconciliation exception queue.

Backend Test Suite **#222 is currently running** against PR #33. Do not merge until the complete gate is green and the final diff has been re-audited.

## Admin Portal settlement realtime — MERGED / VERIFIED

Admin PR #10 corrected restore-time realtime establishment and introduced a global settlement reconciliation boundary. Withdrawal queue, statistics, profit, health and payout-review projections are invalidated from authoritative backend events; socket payloads never become financial truth.

## Flutter withdrawal realtime transport — MERGED / VERIFIED

Flutter PR #17 adds the canonical `withdrawal_progress` / `withdrawal_settled` transport contract to the singleton SocketService. No second socket or feature-owned connection was introduced.

## Business Portal event-contract hardening — MERGED / VERIFIED

Business Portal PR #6 passed its quality gate and was merged after the final duplication audit.

The event layer now classifies backend `BizNotifType` events into projection invalidation domains covering orders, invoices, reservations, transit, Dine-In, KYB/trust, marketing and notifications while preserving the existing singleton socket ownership.

## Current repository state

Merged:

- Backend #29 — financial truth / PoR foundation.
- Business Portal #5 — authenticated transport + realtime token synchronization.
- Backend #30 — dedicated Admin browser sessions.
- Admin Portal #9 — Admin session/realtime integration.
- Backend #31 — provider-settlement truth and reconciliation hardening.
- Backend #32 — provider callback authoritative settlement.
- Admin Portal #10 — Admin settlement realtime reconciliation.
- Flutter #17 — canonical withdrawal realtime transport.
- Business Portal #6 — business event contract hardening.

Open:

- Backend #33 — explicit provider settlement attempt identity.

## Next substantial batches

1. Finish and merge Backend #33 only after the full quality gate and final duplication audit.
2. Replace legacy withdrawal correlation with a direct canonical TransactionHistory reference and durable reconciliation exception queue.
3. Audit all Flutter escrow/order/invoice event payloads against backend emitters and enforce contract tests.
4. Audit Business Portal invoice/reservation/transit/Dine-In mutation → event → refetch paths against actual query keys and backend emitters.
5. Build the Admin reconciliation/exception queue so unresolved financial operations become actionable work rather than dashboard anomalies.
6. Extend the same authoritative-state → domain-event → realtime → client-reconciliation pattern across competition-critical commerce journeys.

## Non-negotiable architecture

`intent → authenticated API → domain authorization → authoritative transaction → durable state → domain event → realtime/notification → client reconciliation`

Backend/domain state remains authoritative. Realtime, caches, analytics and UI projections are downstream. Every financial operation must have an idempotent identity and an observable recovery path.