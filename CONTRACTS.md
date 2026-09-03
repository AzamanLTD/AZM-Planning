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
