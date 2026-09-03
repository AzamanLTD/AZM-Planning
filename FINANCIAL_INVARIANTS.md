# AZAMAN Financial Invariants

## Core rule

No client, portal or generic CRUD path is allowed to become an independent financial engine. The backend financial service is authoritative.

## Required invariants

### Balance

- available balance changes only through authoritative financial operations;
- every balance mutation is atomic with the state that justifies it;
- retries cannot double-credit or double-debit;
- concurrent operations cannot spend the same funds twice;
- ambiguous provider responses have a deterministic reconciliation path.

### Escrow

Conceptual invariant:

`fund: available -= principal + fee; escrowLocked += principal; systemFee += fee`

`release: escrowLocked -= principal; payeeAvailable += principal`

`refund: escrow/dispute locked -= principal; payerAvailable += principal`

Terminal states cannot be re-settled. Database guards are preserved even when service-level idempotency is improved.

### Commerce

- inventory cannot become negative under concurrent checkout;
- order price/variant snapshots preserve what was purchased;
- settlement/refund transitions are monotonic and authority-controlled;
- client estimates are never authoritative monetary outcomes.

### Booking

- overlapping confirmed room reservations are mutually exclusive;
- a seat cannot be confirmed for two passengers;
- holds expire safely and are retry-safe.

### EWA / payroll

- outstanding EWA respects policy limits;
- repayment and payroll amounts are server-authoritative;
- payroll net cannot become invalid/negative;
- consent/disclosure versions are persisted.

## Mutation review checklist

For every financial mutation ask:

1. Who is authorized?
2. What business/tenant owns the target?
3. What is the operation identity/idempotency key?
4. What transaction or conditional write prevents races?
5. Which balances/ledger/history rows change?
6. What happens if the provider times out after commit?
7. When are events emitted?
8. What does the retrying caller receive?
9. How is the result reconciled if an event is lost?
10. What regression test proves the invariant?

## High-priority audits

- wallet balance read→write;
- escrow hold/settle/release;
- trade funding;
- withdrawal approval/rejection/reconciliation;
- order payment/settlement/refund;
- reservation payment/refund;
- dine-in `confirmAndPay`;
- EWA/payroll serializable conflicts;
- privileged Admin force-release/cancel.
