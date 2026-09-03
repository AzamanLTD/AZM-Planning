# AZAMAN State Machines

## Rule

Lifecycle state is domain truth. Generic CRUD must not bypass transitions that have accounting, authorization, inventory, notification or audit side effects.

## Required audit table

For each stateful entity record:

| Entity | Current states | Legal transitions | Illegal/replayed transitions | Side effects | Concurrency guard | Test |
|---|---|---|---|---|---|---|
| Shift | SCHEDULED, CLOCKED_IN, LATE, CLOCKED_OUT, NO_SHOW | dedicated attendance actions | direct status PATCH; terminal replay | hours/wages/no-show/notifications | conditional transition/transaction | required |
| Time off | PENDING, APPROVED, REJECTED | authorized approval/rejection | wrong business, self-approval, replay | audit/notification | conditional update | required |
| Shift swap | pending lifecycle | claim/approve/reject | double claim/approval | schedule changes | conditional/serializable | required |
| Payroll | process/disburse lifecycle | process → authorized disbursement | duplicate disbursement | balances/ledger/history | transaction + idempotency | required |
| EWA | request/approval/repayment lifecycle | policy/permission-controlled | duplicate/replayed withdrawal | balance/ledger/history | transaction + P2034 retry | required |
| Order | canonical commerce lifecycle | authority-controlled transitions | skipped/duplicate settlement | inventory/payment/events | conditional/transactional | required |
| Reservation | hold/confirmed/cancel/refund | date/payment-controlled | overlap/double refund | inventory/payment | unique/conditional/transaction | required |
| Escrow | DRAFT/FUNDED/IN_PROGRESS/PENDING_SETTLEMENT/SETTLED + dispute path | explicit lifecycle actions | terminal replay | locked funds/payee/payer | DB guard + atomic winner | required |

This table is a navigation skeleton; current code and schema must be inspected before marking any row VERIFIED.

## Known shift rule

Attendance transitions are financial/stateful operations. Generic `updateShift()` must not accept `status`; callers must use clock-in, clock-out or no-show actions so wage/attendance side effects cannot be bypassed.

## Concurrency rule

When two requests race for the same transition, one wins deterministically. The loser must either receive a safe conflict or the already-committed canonical state; it must never repeat side effects.

## State-machine test template

For every important transition test:

1. authorized valid transition succeeds;
2. unauthorized actor fails;
3. wrong tenant fails;
4. invalid prior state fails;
5. terminal/replayed request has no duplicate side effects;
6. concurrent requests produce one winner;
7. response represents canonical committed state;
8. event/notification occurs at the correct commit boundary.
