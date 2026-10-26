# AZAMAN Financial Invariants

## Core rule

No client, portal or generic CRUD path is allowed to become an independent financial engine. The backend financial service is authoritative.

## Multi-asset and product-obligation invariants (P0)

Canonical contract: [docs/MULTI_ASSET_MONEY_AND_SETTLEMENT_ARCHITECTURE_2026-10-26.md](docs/MULTI_ASSET_MONEY_AND_SETTLEMENT_ARCHITECTURE_2026-10-26.md).

- Product obligation denomination is explicit, versioned and fixed; domestic obligations default to GHS when the true GHS settlement path is supported. The funding asset never silently changes the promise.
- Currency code and asset identity are separate. USDC network/issuer/contract identity is part of the asset, and GHS/USDC quantities cannot be summed as though they share a unit.
- The wallet may show a combined estimated valuation, but every authorization uses exact asset-specific spendable buckets. Unknown valuation is not zero; a display rate is never an executable quote.
- Cross-asset exchange requires the existing explicit ASSET_CONVERSION ledger identity with exact, per-asset-balanced legs and durable provenance. Do not build a second ledger.
- Cross-currency obligations require a valid, bound, unexpired executable quote, explicit cost disclosure and accepted terms. Saved/demo/stale display rates cannot authorize conversion.
- A pure same-asset operation need not depend on an unrelated FX rate unless its specific risk/limit contract genuinely needs that valuation. FX gates are route-specific and must be documented against all mounted routes.
- Fees and AZAMAN margin are disclosed before consent and recognized from the committed settlement under accounting policy; never skim continuously from customer funds or recognize unrealized revaluation as revenue.
- New GHS-denominated obligations must not be enabled until actual native-GHS funding/liability/withdrawal/reversal and reconciliation are proven. Provider float, UI estimates and scaffolding do not constitute customer GHS assets.
- Susu denomination is fixed before activation; mixed funding must settle into that denomination. A provider-ambiguous contribution is not automatically a member default, and no cycle payout may be duplicated or under-backed.
- Savings goals are not cash; vault/savings locked amounts require actual funds and the correct product currency. Legacy USDC fields/records cannot be silently reinterpreted as GHS.
- Existing ledger, durable operation, route-bound quote, custody and reconciliation authorities must be extended, not duplicated.

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
