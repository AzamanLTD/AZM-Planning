# AZAMAN Cross-Repository Contract Matrix

**Rule:** every important contract has one producer/authority and explicit consumers. A green backend test cannot prove a Flutter/portal contract is compatible.

| Surface | Producer / authority | Consumers | Required verification |
|---|---|---|---|
| Auth/session | Backend | Flutter, Business, Admin | refresh/revoke/expiry/error contract |
| Wallet/money | Backend financial services | Flutter, Admin | balances, mutations, idempotency |
| Retail checkout | Backend | Flutter, Business, Admin | quote/payment/order/inventory contract |
| Escrow | Backend | Flutter, Admin, ticket workspace | lifecycle, ledger, replay/race semantics |
| Orders | Backend | Flutter, Business, Admin | canonical lifecycle + snapshots |
| Invoices | Backend | Flutter, Business, Admin | payment/void/amount authority |
| Dine-in | Backend `dineInService` | Flutter, Business, Admin | tab/add/finalize/pay/close + race |
| Hotel | Backend | Flutter, Business | inventory/booking/amount authority |
| Transit | Backend | Flutter, Business | fare/seat/booking authority |
| EWA/payroll | Backend Business OS | Flutter, Business, Admin | scope, permissions, transitions, money |
| Reservations | Backend | Flutter, Business, Admin | overlap/payment/cancel/refund |
| Dashboard stats | Backend stats contracts | Business, Admin | no client aggregation/fixed truncation |
| Notifications | Backend event/push producers | Flutter, Business, Admin | payload, dedup, action routing |
| Realtime | Backend Socket.IO | all clients | event ordering/filtering/reconnect |
| Experience Blueprint | Backend | Business config + Flutter renderer | schema/version/fallback/published consistency |

## Multi-asset money and settlement contract (P0)

Canonical specification: [docs/MULTI_ASSET_MONEY_AND_SETTLEMENT_ARCHITECTURE_2026-10-26.md](docs/MULTI_ASSET_MONEY_AND_SETTLEMENT_ARCHITECTURE_2026-10-26.md).

| Contract | Producer / authority | Consumers | Required verification |
|---|---|---|---|
| Asset identity and exact buckets | Backend ledger/balance services | Flutter, Admin, relevant Business finance views | Canonical network/token identity; exact available/reserved/locked/pending buckets; no mixed-unit totals |
| Product denomination/contract | Backend Susu, savings/vault and commerce domain | Flutter, Business, Admin | Immutable version/hash, accepted denomination, fee and payout terms |
| Display valuation | Backend valuation read model | Flutter/portals | Native amount vs estimate; rate basis/time; unavailable is not zero; never used to authorize |
| Executable conversion quote | Existing backend route-bound quote/provider authority | Funding Planner + customer client | Source/target exact amounts, asset/currency, route, fees/margin, expiry, recipient binding, one-use acceptance |
| Funding plan / operation | Backend financial operation/orchestration authority | Flutter and domain services | Spendable asset plan, reservation, idempotency, pending/ambiguous recovery, no partial economic residue |
| Ledger conversion | Existing transaction-scoped ledgerService ASSET_CONVERSION | All money domains | Source and destination legs, per-asset balancing, conversion provenance, exact replay |
| Settlement outcome | Backend domain/provider/custody state machine | Flutter, Business, Admin, notifications | Correct finality evidence, refund/reversal and reconciliation; no UI/notification success inference |
| Recipient settlement intent | Backend transfer API + recipient preference | Flutter sender/recipient views | “Send asset amount” vs “recipient receives currency amount”; preference never silently mutates accepted intent |

Rules: local domestic obligations/prices default to GHS only when the corresponding real GHS settlement path exists; user asset holdings remain distinct from product obligations; all cross-currency execution uses a valid quote and explicit consent; same-asset transfer can remain FX-independent; every client is a consumer of canonical backend state.

## Dine-in — first deep contract audit

Trace the exact request/response/event chain for:

1. customer opens tab;
2. customer adds item;
3. business/customer finalizes;
4. payment/`confirmAndPay` executes;
5. invoice/payment/closure commits;
6. socket events reach each consumer;
7. clients recover after missed events.

Explicitly compare Flutter field names and backend expectations (`tabId`, `productId`, `selection`, `quantity`, etc.) and the monetary response fields. Test the race where finalize and payment arrive concurrently.

## Event contract matrix

Known dine-in events requiring producer/consumer verification:

- `dine_in_tab_opened`
- `dine_in_item_added`
- `dine_in_tab_finalized`
- `dine_in_tab_paid`

For each event document: producer, transaction boundary, audience, payload schema, consumer behavior, deduplication, reconnect behavior and HTTP reconciliation fallback.

## Contract status vocabulary

- **VERIFIED:** traced in current main + executable evidence.
- **PARTIAL:** some producers/consumers verified; gaps remain.
- **UNKNOWN:** not yet traced.
- **BROKEN:** mismatch reproduced.

Never upgrade a contract to VERIFIED from discussion alone.
