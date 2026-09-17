# AZAMAN Financial Architecture Investigation — 2026-09-17

**Status:** READ-ONLY architecture investigation. No production mutation, no live provider calls, no financial-data repair, no implementation. This document is the architecture we will implement next.

**Evidence base:** main branches audited 2026-09-17 — `AzamanLTD/AZM-backend` @ `f90f6e4`, `AZM-frontend` @ `0bbd588`, `AZM-businessPortal` @ `0d8a58d`, `AZM-adminPortal` @ `60faed7`, `AZM-Planning` @ `52c3052`. All statements cite actual code paths, not planning snapshots.

---

## A. CURRENT FINANCIAL ARCHITECTURE MAP

### What is REAL (executable against live providers today)
- **Moolre collection (GHS on-ramp rail):** `moolreCollectionService.js` — PIN-push MoMo collection, MOCK/LIVE gated, pure I/O adapter. Quote-backed deposit flow: `quoteFiatDepositController` + `moolreQuoteDepositController` → `TransactionQuote` (server rate, TTL, consume-once, settled-amount match) → webhook credits `User.availableBalance`. Settlement is exactly-once (PR #246).
- **Moolre disbursement + MTN disbursement (GHS off-ramp rail):** `moolreDisbursementService.js`, `mtnDisbursementService.js`.
- **Fiat withdrawal lifecycle:** `finance.service.processFiatWithdrawal` (guarded fiat-pool claim + guarded user debit, deferred economics) → `payoutBatchWorker` (auto/manual dispatch) → `fiatSettlementService` + `providerSettlementAttemptService` (provider evidence) → `withdrawalReconciliationWorker` → `completeFiatWithdrawal` / `reverseFiatWithdrawal`. The most complete money flow in the repo.
- **Rate oracles:** `oracleService` (Kotani `rate/offramp` POST + CoinGecko + fallback FX → `GlobalSettings`), `gatewayService` (Kotani retail/corporate rate pair → `GlobalSettings`).
- **Internal double-entry shadow:** `journalService` + `journalIntegration` (9 call sites: deposits, withdrawals, P2P, trade, peer transfer, escrow lock/release/refund).

### What is SCAFFOLDING (runs in MOCK, cannot execute live)
- **Tatum addresses:** `tatumService.deriveDepositAddress` — xpub-based read-only derivation, `derivationIndex = user.id`, MOCK default. Real code, only exercised in MOCK.
- **Tatum webhooks:** HMAC-SHA512 verification (`x-payload-hash`) is real; subscriptions are fire-and-forget and never persisted. Edge case: if `tatumService` is unbound but a signature header is present in production, the payload passes UNVERIFIED (depositController.tatumCryptoWebhook branches).

### What is FICTION (looks real, is not)
- **On-chain sweeps:** `workers/onchainSweepWorker._executeSweep` is a **placeholder** — no signing, no broadcast. `OnchainSweep` rows are created with status `BROADCASTING`, `txHash: null`, and can never advance. No retry, confirmation, or reconciliation exists. (Balance lookup is real-shaped: Tatum account/balance with the native USDC contract — but the parser also accepts `e.asset === 'USDC'`, which does not distinguish bridged USDC.e.)
- **Crypto withdrawals:** `withdrawalController.cryptoWithdrawal` debits the user, decrements synthetic `SystemHotWallet`, generates a **locally-random fake txHash**, marks the TransactionHistory `COMPLETED`, then POSTs to `api.tatum.io/v3/polygon/transaction` **without `fromPrivateKey` or `signatureId`** — rejected by Tatum's API in all cases. In LIVE mode every crypto withdrawal would debit, fail broadcast, and refund; in MOCK mode it records a fabricated hash as COMPLETED. The refund-on-failure engineering is correct but wrapped around an impossible broadcast.
- **Treasury/cold movements:** `warRoomController.logColdStorage` is audit-trail only (its own comment admits the on-chain movement is out-of-band). `ColdStorageLog` has no source/destination address, tx hash, approval chain, status lifecycle, or confirmation.
- **Exchange:** NO Binance/exchange code exists anywhere — only a marketing string ("Zero-fee Binance withdrawal initiated!" in `walletController`). No models, no client, no order/execution/settlement code. Category: **absent**, not mock.
- **Kotani on-ramp:** none exists. Comments claim "Kotani is still mounted (used for on-ramp / corporate purchases…)" but `gatewayService` implements only the **off-ramp** (rates + MoMo payout + status poll). Model A is greenfield.

### What is SYNTHETIC (DB numbers with no external evidence)
- `SystemMasterCrypto`, `SystemHotWallet`, `SystemFiatPool`, `SystemProfitFees` — four singleton rows (id=1) mutated by service code; bookkeeping opinions, not custody facts.
- **Critical accounting hole:** quote-backed fiat deposits (Moolre + aggregator webhook) credit ONLY `User.availableBalance` — they touch NO system pool. The synthetic "reserve" pool moves only on crypto deposits, corporate purchases, and fiat withdrawals. So every GHS→USDC deposit raises a liability with no matching inventory event; the deposit↔inventory link is operational convention (admins logging corporate purchases), not data.

---

## B. CURRENT DATA-FLOW MATRIX

| Flow | Entry point | Debits | Credits | Provider evidence | Economics recognized |
|---|---|---|---|---|---|
| Moolre GHS deposit | `POST /api/deposit/fiat/initiate/moolre` → webhook | — | `User.availableBalance` (quote.usdcAmount) | Moolre txstatus webhook + `providerRef` | none (no spread/lot) |
| Aggregator GHS deposit | `POST /api/deposit/fiat/initiate` → webhook | — | `User.availableBalance` | shared-secret webhook | none |
| Polygon USDC deposit | Tatum webhook → `processCryptoDeposit` | — | `User.availableBalance` + `SystemMasterCrypto` + `SystemHotWallet` | Tatum webhook (HMAC), txHash idempotency | none |
| Fiat withdrawal | `POST /api/withdrawals/fiat` → `processFiatWithdrawal` | `User.availableBalance` (guarded), `SystemFiatPool` (guarded claim) | `SystemMasterCrypto` increment ("capture") | MTN/Moolre webhook + `ProviderSettlementAttempt` + reconciliation worker | fees + ARBITRAGE_SPREAD **deferred to provider success** (correct) |
| Crypto withdrawal | `POST /api/withdrawals/crypto` | `User.availableBalance`, `SystemHotWallet`; gas→`SystemProfitFees` | — | **fake txHash; live broadcast impossible** | GAS_FEE_REVENUE at request time |
| P2P trade | tradeController/escrow | escrow buckets (guarded) | counterparty + vendor cut | internal | margin logs |
| Corporate OTC purchase | War Room `logCorporatePurchase` | — | `CorporatePurchaseLog` + `SystemMasterCrypto` | screenshot / gatewayReference | implicit |
| Cold movement | War Room `logColdStorage` | — | `ColdStorageLog` row only | **none** | none |
| Profit liquidation | `liquidateProfits` | `SystemProfitFees` (guarded) | `SystemFiatPool` | none | AdminProfitLog row |

---

## C. CURRENT CUSTODY MAP

| Layer | Representation | On-chain/bank evidence? | Verdict |
|---|---|---|---|
| User deposit addresses | `User.tatumPolygonAddress` (single lowercase string) | xpub-derived; no wallet table; subscriptions not persisted | Fragile |
| Master hot wallet | `SystemHotWallet` singleton + `TATUM_TREASURY_ADDRESS` env | No | Synthetic |
| Treasury buffer | none | — | Missing |
| Cold reserve | `ColdStorageLog` intent rows | No | Fiction |
| Exchange liquidity | none | — | Absent |
| Provider balance | `ProviderSettlementAttempt` (off-ramp attempts only) | payout webhooks yes; GHS float no | Partial |
| In-transit | implied by PENDING statuses | partial | Ad hoc |

The database cannot distinguish custody location from customer liability. The PoR service sums `SystemMasterCrypto + SystemHotWallet` (synthetic) against user balances (real liabilities) — it currently counts fictional singleton balances as reserve proof, and its own comments still call the asset "USDT".

---

## D. MODEL A FLOW — Kotani as conversion rail (does not exist; target)

1. QUOTE: backend calls Kotani's on-ramp quote with amountGhs; persists `TransactionQuote` with `route=KOTANI_ONRAMP`, provider rate ID, TTL. (Kotani's exact current on-ramp API surface must be confirmed from their docs — repo evidence covers off-ramp only.)
2. ROUTE DECISION: the quote names Model A explicitly; nothing blends economics.
3. EXECUTION: user pays GHS to Kotani with `receiverAddress` = an Azaman-controlled address; Kotani executes the conversion on-chain.
4. ACTUAL EXECUTION: Azaman observes actual USDC received on the receiver address (webhook + balance check, contract-checked); actual received, not the quote, is authoritative.
5. SETTLEMENT: credit user liability exactly-once on txHash; record Kotani provider reference + rate ID.
6. REALIZED ECONOMICS: realized margin = customer rate − Kotani effective rate, recorded only after settlement; Kotani's fee is execution cost (Kotani is the conversion executor, not merely "the cost").

## E. MODEL B FLOW — Azaman as liquidity provider (exists; needs inventory accounting)

Current: GHS in (Moolre PIN-push, quote-locked rate) → webhook → credit user USDC liability. Separately, admins log corporate OTC purchases (`CorporatePurchaseLog`) incrementing `SystemMasterCrypto`. The two are NOT linked in data.

Target: GHS received → GHS custody/liquidity record → USDC inventory lot selected/created → customer USDC liability created → cost basis attached → realized economics after settlement. The system must never fabricate USDC inventory because a customer deposit succeeded — today it implicitly does (liability up, inventory unchanged, PoR ratio silently drops until an admin logs a purchase).

## F. OFF-RAMP FLOW (target)

The reservation → provider-evidence → settlement spine is sound. Needed: explicit route field (Kotani off-ramp vs Moolre/MTN — currently MTN/Moolre is wired for execution while `gatewayService`'s Kotani payout is a parallel path; unify behind one route decision), actual USDC disposition (custody movement / inventory consumption instead of synthetic master-crypto increment), and the same treatment for crypto off-ramp once real broadcasting exists. Deferred profit recognition on request-only is already correct — keep.

## G. TREASURY ARBITRAGE FLOW (target)

None exists. Target: unencumbered treasury = custody assets − customer liabilities − restricted obligations; arbitrage capital drawn only from that remainder; movements to an exchange are custody movements with lot attribution; P&L on execution, never at quote; strictly separate from customer pricing spread.

---

## H. ENTITY MATRIX (KEEP / EVOLVE / RETIRE)

| Existing entity | Purpose | Problems | Verdict | Target |
|---|---|---|---|---|
| `User.availableBalance/escrowLocked/vendorUnallocated/disputeEscrow` | authoritative customer liability state | mixed into User model | KEEP | liability side of ledger (materialized projection) |
| `TransactionHistory` | mutation log, txHash idempotency, provider refs | overloaded (log + settlement state machine) | EVOLVE | business event + settlement record |
| `JournalEntry` | shadow double-entry | not authoritative; no custody/asset accounts | EVOLVE | **authoritative ledger** |
| `TransactionQuote` | quote-backed deposits | not route-aware; no margin/policy fields | EVOLVE | route, provider rate ID, pricing policy version, expected economics |
| `ProviderSettlementAttempt` | provider evidence (payouts) | off-ramp only, 2 providers | EVOLVE | generalize to ProviderExecution (on-ramp + off-ramp) |
| `CorporatePurchaseLog` | admin OTC purchase log | audit-only; no lots/location/cost basis semantics | EVOLVE | treasury acquisition ledger component (lots) |
| `SystemMasterCrypto/HotWallet/FiatPool/ProfitFees` | synthetic aggregates | zero custody evidence; unsafe as proof | EVOLVE → demote | derived read models (display only) |
| `ColdStorageLog` | intent log | no lifecycle/hash/approvals | RETIRE → replace | `CustodyMovement` |
| `AdminProfitLog` | profit recognition log | negative rows prohibited; coarse sources | EVOLVE | realized-economics record tied to settlement |
| `OnchainSweep` | sweep audit | permanently stuck BROADCASTING; placeholder execution | EVOLVE | custody movement of type SWEEP |
| `CurrencyWallet` | multi-currency user wallets (GHS etc.) | overlaps liability model; unclear authority | AUDIT → likely RETIRE | superseded by liability ledger + GHS rails |
| `AdminApprovalRequest` | 4-eyes approvals | not wired to treasury movements | EVOLVE | approval chain for CustodyMovement |
| Missing: `WalletAddress`, `CustodyAccount`/`CustodyMovement`, `AssetInventoryLot`, `ExchangeAccount/Balance/Order/Execution`, `RealizedEconomics`, `GhsLiquidityRecord`, `ReconciliationRecord` | — | — | CREATE (minimal set; do not create all 16 — reuse above) | |

---

## I. ACCOUNTING JOURNAL EXAMPLES

Chart of accounts: `custody:deposit:usdc` (A), `custody:hot:usdc` (A), `custody:cold:usdc` (A), `custody:exchange:usdc` (A), `custody:provider:usdc` (A), `fiat:momo:ghs` (A), `inventory:usdc:lots` (A, at cost), `user:{id}:liability` (L), `escrow:{id}:locked` (L), `restricted:reserves` (L), `clearing:conversion` (C), `revenue:fees`, `revenue:spread`, `expense:gas`, `expense:provider`, `pnl:inventory`, `pnl:arbitrage`, `equity:treasury`.

1. Direct Polygon USDC deposit (100): D custody:deposit:usdc 100 / C user:42:liability 100.
2. Sweep deposit→hot: D custody:hot:usdc 100 / C custody:deposit:usdc 100 (no liability change; gas: D expense:gas / C custody:hot:usdc).
3. Hot→cold (500): D custody:cold:usdc 500 / C custody:hot:usdc 500.
4. Cold→hot: inverse of 3.
5. Exchange→hot: D custody:hot:usdc / C custody:exchange:usdc.
6. Hot→exchange: inverse of 5.
7. Model A (customer 1,200 GHS → 95 USDC received): D custody:hot:usdc 95 / C user:liability 95 (at actual received); realized margin booked on settlement: D revenue:spread X / C pnl… per policy; Kotani in-band fee: D expense:provider / C custody account it was taken from.
8. Model B (customer 1,200 GHS, quote 12.0 → 100 USDC liability; lot cost 11.7): D fiat:momo:ghs 1,200 / C clearing:conversion 1,200; D inventory:usdc:lots 1,170 / C user:liability 100 + C revenue:spread 30 — spread recognized only at settlement (expected vs realized separated).
9. Fiat off-ramp (100 USDC → 1,200 GHS): D user:liability 100, D fiat:momo:ghs 1,200 / C custody:hot:usdc 100 (inventory at cost), C clearing:conversion 1,200; fees on provider SUCCESS only.
10. External USDC withdrawal: D user:liability 100 / C custody:hot:usdc 100; D expense:gas / C revenue:fees (user-charged gas).
11. Customer fee: D user:liability f / C revenue:fees.
12. Treasury acquisition (1,000 USDC @ 11.9): D inventory:usdc:lots 11,900 / C fiat:momo:ghs 11,900.
13. Treasury sale: inverse of 12 + realized P&L → pnl:inventory.
14. Treasury arbitrage: exchange executions post to custody:exchange:usdc + pnl:arbitrage; never revenue:spread.
15. Provider fee (in-band): D expense:provider / C relevant custody account.
16. Refund/reversal: strict reversal entries against the original transactionId; never negative profit rows.

Semantics: customer balances are LIABILITIES; custody accounts are ASSETS; quoted margin is never booked as revenue until realized; treasury P&L is segregated from customer spread.

---

## J. RESERVE INVARIANTS

REAL USDC ASSETS ≥ CUSTOMER USDC LIABILITIES + RESTRICTED OBLIGATIONS

- Assets counted ONLY from custody accounts with evidence: on-chain balance queries (native-USDC contract 0x3c499c542cef5e3811e1192ce70d8cc03d5c3359) or exchange/provider statements, each with a freshness timestamp; stale evidence (>2h) is excluded.
- In-transit custody counts if a CustodyMovement row is BROADCAST/CONFIRMING with a tx hash.
- Synthetic singletons must NEVER count (this is the exact violation in the current PoR service).
- Disputed escrow = restricted (excluded from available, included in liabilities, labelled).
- The liability Merkle proof stays; the reserve side becomes an attestation snapshot of custody evidence.

---

## K. WALLET ARCHITECTURE COMPARISON (against the actual Tatum account)

| Option | Requires | Fits now? |
|---|---|---|
| **HD wallet + KMS** | Tatum KMS feature; signing via signatureId | **RECOMMENDED if the current Tatum plan has KMS.** Keys never touch the app; matches existing xpub derivation unchanged for deposits; enables real sweeps, withdrawals, hot-wallet ops with audit trails. |
| **Gas Pump addresses** | Tatum Gas Pump feature; contract addresses, gasless sweeps | Viable fallback — removes per-address POL gas funding; replaces xpub-derived deposit addresses with contract addresses (bigger migration). |
| **Virtual Accounts** | — | **EXCLUDED: the current Tatum account carries a Virtual Accounts access restriction; Azaman cannot newly provision it. Do not design against it.** |
| **MPC / Smart Wallets** | higher-tier offering | Not on the current plan; not for v1. |

Verification gate before build: confirm on the actual Tatum account (a) KMS availability, (b) Gas Pump availability, (c) webhook/subscription quota. KMS-first with Gas Pump as documented fallback.

---

## L. RECOMMENDED TARGET ARCHITECTURE (definitive)

1. **Custody (USDC):** keep xpub-derived Polygon deposit addresses, but move them from `User.tatumPolygonAddress` into a durable `WalletAddress` table (userId, network, address, derivationIndex, status ACTIVE/RETIRED, subscriptionId, firstSeen/lastSweep; unique address; one active per user+network; retired never reassigned). Master hot wallet = one KMS-signed address. Cold reserve = Ledger hardware, recorded as a custody account, moved only through the approval lifecycle. **Asset identity:** (network=POLYGON, contract=0x3c499c542cef5e3811e1192ce70d8cc03d5c3359, symbol=USDC) — native only; USDC.e (0x2791B9717a737cA894D692502E875A1a8EAb1cFA) is a distinct asset that current code accepts implicitly (Tatum `currency:'USDC'` on Polygon; sweep parser `e.asset === 'USDC'`) and must be explicitly rejected or separately tracked.
2. **Signing/broadcast:** Tatum KMS signatureId flow for hot wallet + sweeps; the fake-txHash withdrawal path is replaced by real broadcast + confirmation tracking. Gas funded from owned POL custody on the hot wallet (the user-charged USDC gas estimate can stay as a fee, but actual gas comes from owned POL).
3. **Customer liability ledger:** evolve `JournalEntry` into the authoritative double-entry ledger (it already has transactionId grouping, balanced lines, per-account balances). `User.*Balance` columns remain materialized liability projections (guarded conditional writes stay). `TransactionHistory` remains the customer-facing event/settlement record. No parallel ledger.
4. **Model A:** new Kotani on-ramp integration behind the existing quote→execute→settle→realize primitives, with actual-received reconciliation on an Azaman-controlled receiver address.
5. **Model B:** Moolre collection stays; deposits consume/create `AssetInventoryLot`s (evolved from `CorporatePurchaseLog`); realized spread recorded at settlement.
6. **Pricing:** `TransactionQuote` becomes route-aware (route, provider rate ID, pricing policy version, expected economics); ONE rate-sync authority writes `GlobalSettings` (today `oracleService` and `gatewayService` both write the same fields on different cadences — dual-writer hazard).
7. **Treasury/inventory:** lot-based USDC inventory with cost basis, location, realized/unrealized P&L; treasury P&L strictly segregated from customer spread.
8. **Arbitrage/exchange:** exchange abstraction (`ExchangeAccount/Balance/Order/Execution`) behind a kill switch, default OFF; withdrawal whitelist, manual approval, reserve exclusion. Nothing auto-trades.
9. **GHS liquidity:** GHS custody records with states (RECEIVED / AVAILABLE / RESERVED / IN_TRANSIT / PAID_OUT / REVERSED / RECONCILIATION) reconciled against Moolre/MTN settlement evidence; a USDC liability never implies GHS cash without a receipt record.
10. **Movements & 4-eyes:** `CustodyMovement` lifecycle REQUESTED→APPROVED→SIGNING→BROADCAST→CONFIRMING→COMPLETED | FAILED→RECONCILIATION with full field set (source, destination, asset, network, amount, initiator, approvers, signing boundary, request ID, external tx hash, createdAt, broadcastAt, confirmedAt, status, failure reason); approvals via evolved `AdminApprovalRequest`; configurable high-value dual-approval threshold.
11. **Proof of reserves:** liability Merkle (keep) + custody-evidence attestation numerator (new); health = both + journal balance + freshness.
12. **Admin portal:** data contract first, no UI build yet.

---

## M. WHAT TO RETIRE
`ColdStorageLog` (→ CustodyMovement); the fake-txHash broadcast block; `CurrencyWallet` GHS balances after GHS liquidity records land (audit first); one of the two rate-sync writers; "Zero-fee Binance" copy.

## N. WHAT TO EVOLVE
`JournalEntry` → authoritative ledger; `TransactionQuote` → route-aware; `ProviderSettlementAttempt` → ProviderExecution; `CorporatePurchaseLog` → lot component; `AdminProfitLog` → realized-economics record; `OnchainSweep` → custody movement; `System*` singletons → display-only read models; the Tatum webhook unverified-payload edge (tatumService-unbound + header-present) → always verify in production.

## O. WHAT IS MISSING
Real signing/broadcast; wallet address governance; custody accounts & movements; inventory lots & realized economics; Kotani on-ramp; exchange abstraction; GHS liquidity states; route decision in quotes; treasury movement approvals; PoR custody evidence; POL gas custody.

## P. EXACT IMPLEMENTATION ORDER
1. `WalletAddress` table + address governance (replace User-column reads; keep column as mirror). Unblocks everything, low risk.
2. KMS decision + real withdrawal broadcast + sweep execution (verify KMS availability first; the fake-hash path is the biggest live financial-integrity risk in the repo).
3. CustodyAccount/CustodyMovement + journal asset accounts → PoR numerator evidence-based; demote System singletons to display.
4. Ledger authority migration (JournalEntry authoritative for conversion/arbitrage/inventory; guarded column writes stay).
5. Model B lots + realized economics (link deposits↔inventory; realized spread at settlement).
6. Quote route-awareness + single rate authority.
7. Kotani Model A on-ramp behind the same primitives.
8. GHS liquidity states + reconciliation records.
9. Treasury movement approvals (4-eyes) + admin data contracts.
10. Exchange abstraction (flag OFF) + arbitrage P&L.
11. Admin portal Treasury UI (build last, from the contract).
12. Regulatory gating review + kill-switch audit.

## Q. RISKS / OPEN DECISIONS
- Tatum KMS / Gas Pump availability unconfirmed (verify before step 2).
- Kotani's current on-ramp API surface unknown (repo evidence covers off-ramp only).
- `derivationIndex = user.id` couples user identity to key space; keep (already-derived addresses) but record the index in WalletAddress and allow later rotation.
- Moolre GHS settlement reconciliation source (settlement-report endpoint vs manual attestation) undecided.
- USDC.e: reject at deposit vs separately track (recommend reject + explicit asset identity everywhere).
- Legal/compliance confirmation required before live activation of: custodial wallets, customer USDC custody, GHS↔USDC conversion both directions, stablecoin dealing, exchange/trading, arbitrage, reserve policy, AML/KYC depth, transaction monitoring, custody controls. No legal conclusions drawn here.

## R. PRODUCTION SAFETY GATES
1. No withdrawal/broadcast may be COMPLETED without a provider/chain terminal evidence row.
2. No custody movement without a CustodyMovement row reaching terminal state; no synthetic balance counts as reserve evidence.
3. Per-feature kill switches (on-ramp, off-ramp, crypto withdrawal, sweeps, exchange, arbitrage) in GlobalSettings, default OFF, admin-gated.
4. Four-eyes above a configurable threshold for any custody movement.
5. Reconciliation freshness SLOs (evidence >2h old excludes from reserves; alert at 30m).
6. Full regression suites (deposit/P2P settlement, PoR integrity) green at exact head before merge.

---

## §14 — Admin Treasury UI data contract (no build yet)
Sections: Treasury Overview (custody vs liabilities vs restricted, reserve ratio + evidence freshness); Custody (accounts, addresses, balances + last attestation); Liquidity (GHS states, fiat pool, Moolre/MTN settlement evidence); Pricing (policy versions, current rates + sources, route map); Conversion Routes (Model A/B toggle + health); Provider Executions (Kotani/Moolre/MTN attempts); Exchange Liquidity (flagged-off); Hot Wallet (sweep queue + POL gas balance); Cold Reserve (movements + approvals); Reconciliation (exceptions, stale evidence); Reserves (PoR snapshots + attestation); P&L (fees, realized spread, inventory P&L, arbitrage P&L — segregated); Approvals; Audit. Every screen reads ledger/custody read models — no client-side arithmetic.

## §15 — Frontend findings
- `currency_provider.dart` already prefers canonical `liveRetailRate` with legacy fallbacks (correct direction).
- `withdrawal_screen.dart` labels balances USDC (comment records the USDT→USDC rename); the deposit flow is GHS-denominated with server quotes.
- No client-side authoritative rate arithmetic found on the audited paths; remaining work is removing residual display-only hardcoded conversions during quote route-awareness.

**Bottom line:** the liability side, quote/deposit/off-ramp settlement spine, and provider-evidence layer are genuinely solid. The fatal gaps are all on the asset side: no real signing/broadcast, no custody evidence, synthetic reserves, no inventory/realized economics, and no Model A. Implement in §P order, KMS-first, with Virtual Accounts excluded.
