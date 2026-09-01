# Financial Realtime Contract Audit — 2026-09-01

## Scope

Audited the Backend producers for the highest-risk financial realtime events and traced their current downstream socket consumers in Flutter and Business Portal, with Admin Portal room conventions checked separately.

The purpose is to keep executable tests aligned with actual producer semantics rather than forcing different event families into a common payload.

## Verified escrow contract

`services/escrowService.js` emits `escrow_funded` after the atomic funding transaction completes.

Rooms:

- `user_<payerId>`
- `user_<payeeId>`
- `admin_spy_room`

Payload fields currently emitted:

- `escrowId`
- `ticketId`
- `status` (`FUNDED`)
- `amountUsdc`
- `payerId`
- `payeeId`
- `fundedAt`

`escrow_settled` follows the same three-room targeting and carries:

- `escrowId`
- `ticketId`
- `status` (`SETTLED` for the normal settlement event)
- `amountUsdc`
- `payerId`
- `payeeId`
- `settledAt`

The release path deliberately does not emit `escrow_settled` when the final status is `RELEASED`. `RELEASED` is the dispute-resolution outcome and remains a distinct state/event path.

Flutter's `SocketService` registers both `escrow_funded` and `escrow_settled` through the singleton escrow listener registry. This confirms the events are convergence signals consumed downstream rather than a second financial authority.

## Verified withdrawal contract

The Backend withdrawal webhook has a different and narrower contract than the escrow events.

For pending provider callbacks, `withdrawal_progress` is sent only to:

- `user_<userId>`

Payload fields:

- `reference`
- `status` (`PENDING`)
- `stage` (`PROCESSING`)
- `label`
- `pct`
- `timestamp`

For terminal callbacks, the Backend emits `withdrawal_progress` and then `withdrawal_settled`, both to `user_<userId>`.

`withdrawal_progress` terminal payload adds `providerTxId` and uses `stage`/`pct` appropriate to `COMPLETED` or failure.

`withdrawal_settled` payload fields are:

- `reference`
- `status`
- `providerTxId`
- `changed`
- `timestamp`

Admin notification is intentionally a separate projection:

- room: `admin_spy`
- event: `admin_alert`
- `type`: `WITHDRAWAL_SETTLED` or `WITHDRAWAL_FAILED`
- `reference`
- `status`
- `providerTxId`
- `changed`
- `timestamp`

This differs from the earlier proposed `admin_spy_room` / `withdrawalId` shape. The implementation and executable contract tests now preserve the actual Backend architecture rather than inventing a second Admin withdrawal event.

## Executable verification

Backend PR #72 adds `__tests__/financial-realtime-contracts.test.js` covering:

- exact escrow room targeting and payload keys;
- exact settled-vs-released semantics;
- pending withdrawal progress targeting and schema;
- terminal withdrawal progress + settlement projections;
- separate Admin `admin_alert` projection.

The tests use mocked persistence/socket boundaries and therefore remain contract tests rather than integration tests.

## Architectural rule

Backend/domain state remains authoritative. Socket messages only signal convergence. Consumers must refetch/reconcile canonical API/query state rather than treating the event payload as a financial ledger.

## Follow-up

After PR #72 passes the full PostgreSQL-backed Backend gate, merge it and then continue the contract audit into any uncovered financial event families and Flutter marketplace dead-code review. Keep event tests synchronized whenever producer payloads or room topology changes.
