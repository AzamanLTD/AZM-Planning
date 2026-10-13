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
| Marketplace discovery & mix | Backend (contract: `MARKETPLACE_MIX_REDESIGN_2026-10-10.md`, §1–§9) | Flutter, Business | taxonomy/eligibility gates, ranking inputs, storefront kill-switch semantics |
| GHS listing price (`BusinessPrice`) | Backend (contract: `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md`) | Flutter, Business, Admin | price scope/date model, effective resolution, backfill no-inference policy, eligibility |
| Merchant GHS payable | Backend (contract: `MERCHANT_GHS_PAYABLE_FINANCIAL_CONTRACT_2026-10-10.md`, **REV 2.1 canonical**) | Flutter, Business, Admin | payable posting balance, fee bearer, payout correlation record, refund paths |
| GHS price backfill (P2) | Backend (plan: `MARKETPLACE_GHS_PRICE_BACKFILL_WORKLIST_PLAN_2026-10-11.md`) | Admin tooling | coverage states, provenance rules, guarded create-if-absent, race contract |

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

## Marketplace — GHS economy canonical versioning (2026-10-13)

One canonical version per document; superseded text is corrected in place, never left ambiguous:

- **Marketplace mix architecture:** `docs/MARKETPLACE_MIX_REDESIGN_2026-10-10.md` — D-1 corrected in place (GHS canonical listing currency; the earlier USDC-default D-1a is explicitly superseded in the header). Decisions D-2..D-8 finalized per §9.
- **GHS pricing & payment:** `docs/MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md` — the single price-model authority (`BusinessPrice` scope/date model, exact 2-dp pesewas, no USDC inference, drafts ineligible).
- **Merchant GHS payable:** `docs/MERCHANT_GHS_PAYABLE_FINANCIAL_CONTRACT_2026-10-10.md` — **REV 2.1 is canonical** (immutable business identity §6.1, platform payout correlation record). REV 2 and earlier are superseded; the revision header in the document is the version register.
- **Backfill worklist (P2):** `docs/MARKETPLACE_GHS_PRICE_BACKFILL_WORKLIST_PLAN_2026-10-11.md` — planning only; blocked on PR #334 lead review; merchant/backfill race contract per its latest correction.

Implementation status lives in the repos, not here: price foundation PR AzamanLTD/AZM-backend#334, discovery-eligibility PR #333, receive/request provisioning PR #339 — all open, none merged at the time of this entry.
