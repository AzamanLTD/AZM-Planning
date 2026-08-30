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

### Remaining financial identity issue

The legacy `Withdrawal` model still lacks a dedicated provider-attempt/reference relation. Reconciliation therefore retains a legacy correlation mechanism. The next backend financial batch should introduce explicit durable provider-attempt identity and reconciliation exceptions instead of relying on timestamp/amount matching.

## Admin Portal settlement realtime — MERGED / VERIFIED

Admin PR #10 corrected restore-time realtime establishment and introduced a global settlement reconciliation boundary. Withdrawal queue, statistics, profit, health and payout-review projections are invalidated from authoritative backend events; socket payloads never become financial truth.

## Flutter withdrawal realtime transport — MERGED / VERIFIED

Flutter PR #17 adds the canonical `withdrawal_progress` / `withdrawal_settled` transport contract to the singleton SocketService. No second socket or feature-owned connection was introduced.

## Business Portal event-contract hardening — IN PROGRESS / PR #6

A repository-wide research pass covered the Business Portal socket singleton, notification hook, query-key factory, invoice consumer and backend `BizNotifType` surface.

### Defect found

The portal listened to `biz_notification`, but only order events triggered downstream cache invalidation. Backend business events also cover invoices, reservations, transit, Dine-In, KYB/trust and marketing. Those mutations could therefore reach the portal without refreshing the affected projection.

### Implementation in PR #6

- Centralized business event classification.
- Invoice event invalidation for both current and legacy query roots.
- Reservation projection invalidation.
- Transit booking/trip projection invalidation.
- Dine-In projection invalidation.
- Trust/business-profile invalidation.
- KYB/follower business-profile invalidation.
- Marketing/ad projection invalidation.
- Notification cache refresh remains global.
- Existing singleton socket and listener teardown are preserved.
- Regression test proves `INVOICE_PAID` refreshes invoice projections and listener cleanup occurs on unmount.

The implementation intentionally uses invalidation rather than copying financial/business payloads into client state: backend REST/domain state remains authoritative.

**PR #6 remains open until Business Portal CI is green and its final diff is re-audited against `main`.**

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

Open:

- Business Portal #6 — business event contract hardening.

## Next substantial batches

1. Finish and merge Business Portal #6 only after its full quality gate and final duplication audit.
2. Replace legacy withdrawal correlation with explicit provider-attempt identity and durable reconciliation exceptions.
3. Audit all Flutter escrow/order/invoice event payloads against backend emitters and enforce contract tests.
4. Audit Business Portal invoice/reservation/transit/Dine-In mutation → event → refetch paths against actual query keys and backend emitters.
5. Build the Admin reconciliation/exception queue so unresolved financial operations become actionable work rather than dashboard anomalies.
6. Extend the same authoritative-state → domain-event → realtime → client-reconciliation pattern across competition-critical commerce journeys.

## Non-negotiable architecture

`intent → authenticated API → domain authorization → authoritative transaction → durable state → domain event → realtime/notification → client reconciliation`

Backend/domain state remains authoritative. Realtime, caches, analytics and UI projections are downstream. Every financial operation must have an idempotent identity and an observable recovery path.