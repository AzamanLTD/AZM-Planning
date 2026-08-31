# AZM Master Architecture & Execution State

> Continuity document for multi-session engineering. This is a current-state control document, not a replacement for the historical master roadmap.

## System objective

AZM is treated as one distributed product, not five independent repositories. The authoritative flow is:

**Admin Portal / Business Portal / Flutter → Backend authority → committed mutation → realtime convergence signal → canonical client projection.**

Realtime payloads are signals, not financial truth. Clients must refresh authoritative projections after financial/state mutations.

The detailed cross-portal realtime contract is maintained in `docs/REALTIME_CONTRACT.md` and must be consulted before changing an event, listener, subscription, or invalidation path.

## Repository roles

- **AZM-backend:** authoritative domain, ledger, transaction boundaries, state transitions, realtime event production, reconciliation.
- **AZM-businessPortal:** business operating interface; consumes backend events and invalidates canonical queries.
- **AZM-adminPortal:** privileged control plane; must never bypass backend transactional invariants.
- **AZM-frontend:** customer Flutter application; one realtime singleton; canonical REST/provider convergence.
- **AZM-Planning:** continuity brain: roadmap, architectural contracts, implementation ledger, discovered hazards, and completed work.

## Current verified completed work

- Atomic invoice payment claim/replay protection is merged in Backend.
- Withdrawal worker canonical transaction identity, atomic claiming, PROCESSING recovery, network propagation, and provider-label compatibility are merged in Backend.
- Flutter invoice-paid convergence is merged.
- Flutter invoice-received convergence is merged.
- Flutter order-delivered convergence is merged.
- Flutter customer order → existing TicketWorkspace escrow funding entrypoint was implemented in PR #30, passed its exact-head Flutter Quality gate, and was merged to Flutter main as `3c2a2f9e6f8a700ec211d636e0048fd1005a74db`.
- Planning contains the cross-session architecture/execution control document and the cross-portal realtime contract.

## Non-negotiable engineering invariants

1. Never introduce a second socket for a capability already served by the singleton transport.
2. Never make a socket payload authoritative for money, invoice totals, order state, escrow state, or balances.
3. Every financial mutation must have one authoritative backend transaction boundary.
4. Every concurrent state transition must use a conditional/atomic claim or database invariant.
5. Every post-commit event must be emitted only after the authoritative mutation commits.
6. Every new event must be checked against all existing producers and consumers before implementation.
7. A retry must converge to the committed canonical state rather than create a second financial mutation.
8. Open PRs must be compared against current main before new work is started; superseded work must be closed rather than duplicated.
9. A task is not complete merely because a PR is green: after merge, search the whole system for duplicate implementations and stale consumers.
10. Planning must be updated after meaningful architectural discoveries so a future session can continue without guessing.

## Active high-risk boundaries

### Escrow satisfaction concurrency

`escrowService.markSatisfied` atomically claims each participant's satisfaction flag and the database migration has a terminal-state trigger preventing a stale `PENDING_SETTLEMENT` write after settlement. The remaining improvement is response convergence: a concurrent losing request must be able to return the already-committed canonical settlement rather than presenting a misleading failure, without weakening the database trigger or adding a second settlement path.

Before implementation, inspect:

- `services/escrowService.js`
- `controllers/escrowController.js`
- SmartEscrow migrations/triggers
- escrow service/controller tests
- Flutter `EscrowService`, `EscrowNotifier`, `EscrowStatusPanel`, and TicketWorkspace listeners
- Business Portal escrow event/query invalidation
- all `escrow_settled` producers/consumers

### Admin force release — Backend issue #48

The Admin force-release flow has an intermediate `DISPUTED → PAID` boundary before `completeTrade`. A naïve catch rollback is unsafe because another actor may advance the trade after the intermediate write. The tracked implementation is Backend issue #48. The correct implementation should make the operation atomic or use a conditional rollback that cannot overwrite a newer committed state.

Before implementation, inspect:

- `controllers/adminController.js`
- `services/p2p.service.js`
- Admin action tests
- all `completeTrade` callers
- balance/ledger writes
- TransactionHistory/AdminProfitLog writes
- post-commit events
- Admin Portal action UI and realtime response handling

### Order webhook contract — Backend issue #49

The current business order service uses `PAID → DELIVERED` while the controller has historically emitted `order.completed` from the delivery endpoint. The webhook dispatcher documentation indicates `order.delivered` and `order.completed` are separate semantic events. Do not add another event until every producer/consumer is mapped. The tracked implementation is Backend issue #49.

Before implementation, inspect:

- `services/businessOrderService.js`
- `controllers/businessOrderController.js`
- webhook controller/dispatcher/emitter
- Business Portal webhook configuration/consumers
- Admin consumers if any
- Flutter order event consumers
- order/escrow transition tests

### Ticket-scoped realtime subscriptions

Flutter TicketWorkspace currently needs ticket-scoped subscriptions while the singleton socket layer owns global escrow dispatch. Replacing raw listeners with a single callback would lose independent ticket subscriptions. The correct future architecture is a multi-subscriber registry with exact unsubscribe handles, ticket filtering, reconnect safety, and no second socket.

Before implementation, inspect all raw socket listeners, singleton registrations, lifecycle cleanup, reconnect behavior, and tests.

## Active issue register

- **Backend #48 — Admin force-release atomicity:** research complete enough to identify the unsafe intermediate PAID state; implementation must unify the financial transaction boundary rather than add rollback hacks.
- **Backend #49 — Order webhook semantics:** producer/consumer mapping required before any event rename/addition; avoid duplicate delivery/completion events.
- **Escrow satisfaction convergence:** remains the next concurrency implementation candidate after full affected-file verification.

These issues are intentionally tracked as engineering work, not as permission to implement without the required research pass.

## Session continuation protocol

At the beginning of every new engineering session:

1. Read this document and the historical master roadmap.
2. Read `docs/REALTIME_CONTRACT.md` before touching realtime behavior.
3. Enumerate open PRs across all repositories.
4. Compare each open PR against current main and close/replace stale duplicates.
5. Identify the highest-risk incomplete boundary.
6. Research every producer, consumer, database invariant, UI projection, and test affected by that boundary.
7. Implement the smallest canonical change.
8. Run exact-head CI.
9. Merge only when the repository gate is green and the change is not duplicated elsewhere.
10. Re-research current main after merge.
11. Update this document with completed work and newly discovered risks.

## Current PR state at the time of writing

No implementation PR from the current AZM engineering loop is intentionally left open. Flutter PR #30 is merged. Backend issues #48 and #49 are tracked work items; they are not duplicate PRs.

## Quality standard

The target is not merely "works now." The target is deterministic behavior under retries, concurrent requests, reconnects, stale clients, partial failures, and multi-portal operation. Any implementation that creates a second source of truth, duplicate event, duplicate listener, or ambiguous transaction boundary is considered incomplete even if its happy-path tests pass.
