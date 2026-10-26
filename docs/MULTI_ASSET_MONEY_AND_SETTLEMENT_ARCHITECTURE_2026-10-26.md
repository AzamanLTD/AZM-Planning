# AZAMAN Multi-Asset Money, Funding and Settlement Architecture

**Decision date:** 2026-10-26  
**Status:** ACTIVE TARGET ARCHITECTURE — implementation must conform; open provider, regulatory and legacy-contract facts remain explicit gates  
**Priority:** P0 — financial correctness  
**Applies to:** AZM-backend, AZM-frontend, AZM-businessPortal, AZM-adminPortal and all future fiat/virtual-asset corridors  
**Canonical plan:** ROADMAP.md, Wave 12  
**Purpose:** Establish one durable money model for wallets, mixed-asset funding, Susu, savings, vaults, peer transfers, cross-border transfers, marketplace, restaurant/dine-in checkout, withdrawals, and future currencies.

> This document describes the target contract, not proof that every path already implements it. It does not authorize merges, deployments, provider calls, production-data changes, or the activation of a real-money route. Existing contracts and balances must be audited before migration; never relabel historical USDC fields as GHS.

---

## 1. Executive decision

AZAMAN is a multi-asset financial system, not a USDC wallet with GHS labels pasted on top.

**Domestic obligations should default to GHS. The asset used to fund an obligation must not silently determine the currency of the obligation.** A member may own USDC and use it to fund a GHS Susu contribution, but the contribution and the GHS payout promise stay denominated in GHS once the applicable conversion and settlement conditions are met.

Build one reusable **Funding Planner + Quote and Settlement Orchestrator** around the existing authoritative double-entry ledger, existing durable operation/idempotency facilities, existing quote authority, and existing provider/custody state machines. Do not build a parallel ledger or make the frontend an FX or accounting authority.

A screen may show an estimated combined wallet value, but a transaction must reason over exact, asset-specific spendable amounts and an explicit recipient/product obligation. The combined display value is not itself an account balance that can be debited.

### Non-negotiable target

Every financial journey must be able to answer, unambiguously:

1. **What does the customer owe or expect to receive?** Exact amount, denomination currency, product-contract version and recipient.
2. **What asset does the customer actually own and have available to spend?** Exact amount, canonical asset identity, network where relevant, and available/reserved/locked/pending status.
3. **How will the source asset satisfy the obligation?** Direct same-asset transfer, a real conversion quote, or a declared settlement route.
4. **Who bears conversion, provider and network costs?** Disclosed before authorization, with any platform margin explicit.
5. **What proves the transaction completed?** The required committed ledger, domain state and provider/chain/fiat settlement evidence, followed by reconciliation.

No product may invent its own answer to these questions.

## 2. What is verified now, and what is not

This was reconciled against live GitHub on 2026-10-26. The current backend main source inspected for the ledger boundary is commit 0edefa1dbe056343080f2d762d89a1b0936c109c.

### Existing foundation to preserve

- The main backend has an authoritative, caller-transaction-scoped ledger service with exact Decimal validation, asset identity on ledger accounts, per-asset-balanced posting rules, and an explicit ASSET_CONVERSION boundary. Extend it; do not replace it with a new ledger.
- The Planning execution ledger records the prior P4 liability-ledger and P5-A multi-asset identity boundaries as completed, and P5-B inventory lots, P5-C route-aware quotes and P5-D evidence-backed GHS liquidity as completed. Re-read the actual current main and evidence before implementing each consumer; older Planning state files contain stale snapshots.
- Route-bound quotes, durable financial operation/idempotency mechanisms, custody evidence, provider settlement evidence and reconciliation patterns already exist. Reuse the canonical implementations rather than creating parallel versions.
- Moolre remains the current contracted external fiat rail in the existing product-direction contract. A mobile-network label is not proof of a separate provider contract.

### Current gaps that matter to this plan

- Mainline Susu models still contain USDC-specific fields such as SusuGroup.contributionUsdc and SusuContribution.amountUsdc. Mainline vault models/services still use targetAmountUsdc and currentAmountUsdc. These fields evidence current implementation semantics; they do not prove that USDC denomination is the correct default for new domestic products.
- Backend PR #367 (native-GHS customer wallet, FX snapshot and balance valuation), #368 (universal FX gate), and #369 (Susu payout/shared-vault integrity) were open and unmerged at review time. They are not production truth simply because their CI is green.
- GHS liquidity/off-ramp accounting is not the same thing as a fully usable native-GHS customer wallet. The open #367 contract says the native-GHS customer funding/debit path is not yet wired into the existing Moolre flows. Verify the exact mainline funding, debit, payout and reconciliation path before treating GHS as spendable.
- Frontend PR #203 was open and unmerged at review time. Its exact review-round-2 head was 13f4f8ea8b05e6dfd0bea5da4c10dd9efce545ca; Flutter Quality and Android Integration succeeded on that head. Its safer parsing and unknown-state behavior are useful, but a correct UI mapper cannot resolve a product/ledger denomination mismatch.

References: [backend ledger service](https://github.com/AzamanLTD/AZM-backend/blob/0edefa1dbe056343080f2d762d89a1b0936c109c/services/ledgerService.js), [Susu schema on main](https://github.com/AzamanLTD/AZM-backend/blob/0edefa1dbe056343080f2d762d89a1b0936c109c/prisma/schema.prisma), [PR #367](https://github.com/AzamanLTD/AZM-backend/pull/367), [PR #368](https://github.com/AzamanLTD/AZM-backend/pull/368), [PR #369](https://github.com/AzamanLTD/AZM-backend/pull/369), [frontend PR #203](https://github.com/AzamanLTD/AZM-frontend/pull/203).

## 3. Vocabulary: keep these concepts separate

| Concept | Meaning | Example | Must not be confused with |
|---|---|---|---|
| Currency | A denomination such as GHS, USD or NGN | GHS 100.00 | A specific blockchain asset/network |
| Asset identity | The canonical asset actually held/transferred, including network/issuer/contract where relevant | Native Polygon USDC, not USDC.e | The generic label “USDC” |
| Asset holding | Exact units of an asset attributed to a user or to the platform | 50.000000 USDC on the supported network | Its changing GHS estimate |
| Product obligation | The amount/currency the product contract requires or promises | GHS 100 contribution | The customer's source asset |
| Display valuation | A marked estimate translated into another currency using a named observation | 50 USDC ≈ GHS X | Cash, a quote, a guarantee or spendable balance |
| Executable quote | A route-specific offer with exact source and target amounts, costs, expiry and provider identity | Debit X USDC to deliver GHS 30 net | A general FX observation or UI estimate |
| Funding plan | The planned combination of existing spendable assets and confirmed conversion/settlement steps | GHS 70 plus a quoted USDC conversion for the GHS 30 shortfall | A completed payment |
| Settlement result | The evidence-backed outcome of the requested economic movement | Provider confirms the required GHS payout | Request accepted, spinner completed, or a webhook without required verification |
| Customer liability | What AZAMAN owes the user in an asset/currency according to the authoritative books | 50 USDC owed | Synthetic treasury balance or a display estimate |
| Restricted amount | Amount reserved, locked, escrowed, pending, disputed or otherwise not spendable | USDC reserved for a pending payout | Available balance |
| Realized platform revenue | Fee/spread supported by a committed, settled transaction and ledger evidence | Agreed conversion margin at settlement | Unrealized FX revaluation of customer assets |

### Currency versus asset

Use a currency code for a product obligation (for example GHS). Use a separate canonical asset identifier for what is actually held or moved. USDC on Polygon and USDC on Stellar are not interchangeable assets merely because both display “USDC.” Native USDC and bridged USDC.e are distinct. Network, token contract/issuer and supported custody route must be known for on-chain activity.

Never add a number denominated in GHS to a number denominated in USDC. Convert only through an explicit, evidenced conversion event. Never use a JS floating-point projection as the authoritative quantity.

## 4. Decisions to apply across the product

These are the recommended target decisions. They should be implemented once as shared rules, not re-decided by each feature. External provider/legal dependencies and legacy data semantics remain gates as described below.

### D1 — Domestic obligations default to GHS; product denomination is explicit and fixed

- New domestic Susu groups, local savings commitments, domestic goal/vault products and local retail/restaurant prices default to GHS denomination where the product is intended to promise domestic fiat value.
- The denomination is pinned in a versioned, accepted product contract before activation/funding. A running Susu, vault or committed savings product cannot silently change currency between cycles or after users have funded it.
- An explicitly USDC-denominated product is allowed as a distinct product choice. Its monetary promise is in USDC; a GHS equivalent is an estimate and must be marked as such.
- Do not activate a “GHS-backed” product promise until the backend has a real, tested and reconciled path that funds, reserves, debits, pays out and reverses the corresponding GHS liability. A GHS display, market rate, GHS price field, provider float or PR is not proof of a customer GHS wallet.
- Existing legacy USDC-denominated records retain their historical meaning until audited. No bulk relabel, implicit conversion, balance rewriting or contract reinterpretation.

### D2 — Reject majority-driven/dynamic Susu denomination

Do not set or change a Susu group's contractual currency from a rolling 50/50 or 60% membership/funding threshold. That would make the promised payout change with membership, creates gaming and disputes, and can expose existing members to a different obligation without informed consent.

Instead:
- The creator selects GHS or an explicitly USDC-denominated contract before activation. GHS is the default for a Ghana-domestic group.
- The app may recommend a denomination before activation, using declared/consented participant funding or payout preferences. Do not inspect or expose private wallet balances merely to classify a group. Any percentage threshold is only a transparent UI recommendation heuristic, never a financial rule.
- Show the exact obligation currency and payout promise before every participant accepts the contract.
- On a GHS contract, the contractual contribution and net payout are fixed in GHS. The amount of USDC required to fund a contribution can vary by transaction quote, but after the required GHS amount is actually funded the group is not exposed to an unconverted USDC holding for that contribution.
- On an explicitly USDC contract, the contribution and payout are fixed in USDC. A GHS equivalent can be shown as “≈” throughout the lifecycle; it is never promised GHS.
- A group with mixed funding assets is valid only if all contributions are converted/settled into the group's fixed denomination according to the accepted terms. “Members paid in different assets” must not make the cycle itself a numerically mixed-asset pot.

### D3 — A shared Funding Planner reasons from real asset buckets

The Funding Planner is the common decision layer for Susu contributions, eligible savings/vault deposits, wallet transfers, retail checkout, restaurant/dine-in, and supported withdrawals.

Inputs:
- authenticated payer and authorized action;
- the exact obligation amount and denomination;
- canonical asset-level balances;
- available, reserved, locked, pending, disputed and restricted states;
- allowed funding assets/routes and applicable risk/limits;
- any user-selected funding preference or bounded standing instruction.

Default for a domestic GHS obligation:
1. inspect the exact spendable GHS balance;
2. use GHS first under the documented funding preference;
3. if it is insufficient, show the shortfall and offer eligible source assets such as USDC;
4. obtain a current executable quote for the shortfall (including provider fees, AZAMAN margin and net target proceeds);
5. reserve funds with a durable single-winner operation identity and execute only under accepted terms;
6. mark the contribution/order/payment funded only when the relevant rail's settlement rules prove the required target amount is secured.

The planner may show a combined “estimated total value” for display. It must never convert that display into a debit authority. A UI control should let users select the source asset and target/recipient currency where relevant. Avoid interrupting users when the default is unambiguous, but expose the option and make the choice explicit when conversion, shortfall, unusual amount or a new route changes the economic terms.

A mixed-asset funding plan must not partially leave the user charged if the corresponding product operation fails. Reserve/execute/release/reconcile rules must be defined for the provider saga; never pretend an external provider action is rolled back by a database rollback.

### D4 — Conversion quote is separate from a display rate

A display-rate observation is for an explicitly approximate valuation. It must not be used as an executable price.

An executable quote must, to the extent the supported provider/route supplies these concepts, pin:
- quote ID and one-use identity;
- source asset/currency and exact source amount;
- target asset/currency and exact guaranteed or minimum target proceeds;
- canonical route/provider and recipient/destination identity;
- quoted rate and quote basis;
- provider charges, network costs, AZAMAN fee/margin and total costs;
- fees' payer and resulting net recipient proceeds;
- minimum/maximum bounds, issue time, expiry and accepted terms/version;
- status and a durable operation/idempotency identity.

A quote is accepted at most once. Changed amount, recipient, route, provider, currency, network, fee/terms or expired quote requires a new quote and confirmation. The exact source/target arithmetic and rounding/quantization rules are server-authoritative and tested at the asset/rail boundary.

For a fixed-price transaction, execute only on a valid, unexpired accepted quote and the provider guarantees documented by that route. Do not invent a quote guarantee if the provider only returns an estimate.

**Do not adopt an arbitrary universal “2% tolerance.”** It is not grounded in a provider contract or approved risk policy. For recurring/standing conversions, store explicit, user-approved execution bounds such as maximum source debit, minimum target proceeds, maximum all-in fee, and (where needed) maximum effective spread/deviation. If any bound is breached or the provider cannot produce a valid quote, skip that occurrence and notify the user; do not silently worsen the economics. If the UI cannot show or the backend cannot enforce the bound, automatic conversion is not enabled.

Saved/demo/stale FX observations can support appropriately labelled display-only estimates if policy permits. They never authorize a conversion or fulfill a GHS-denominated obligation.

### D5 — Transfers distinguish “send this asset” from “deliver this amount”

Offer two explicit money intents, generalized for future GHS, NGN, USD and supported virtual assets:

1. **Send amount in asset** — for example, send 50 USDC. The recipient receives the specified supported asset amount (net of any explicitly shown network/transaction costs). No implicit conversion occurs.
2. **Recipient receives amount in currency** — for example, recipient receives GHS 100. The sender funds an obligation denominated in that target currency; the Funding Planner gets a real route-specific quote and explains the source debit/cost before confirmation.

Recipient account preferences may preselect a settlement currency and reduce friction, but must not silently override an amount/asset the sender explicitly entered. The preferred settlement currency is a default UX/recipient setting, not permission to mutate the transaction contract without the sender's confirmation. Sender bears conversion costs by default when they choose a target-currency delivery; any alternative allocation must be disclosed and deliberately selected.

Represent recipient preferred settlement currency as a future-proof account preference, not hardcoded “GHS everywhere.” The currently supported local fiat currency can default to GHS, but the architecture must not require rewrites when NGN or other corridors are supported.

### D6 — Marketplace, restaurant and dine-in are locally GHS-denominated by default

- Domestic listed prices, order-price snapshots and merchant proceeds should be GHS-denominated when that is the merchant contract.
- A USDC-funded customer may pay by converting the necessary amount using an accepted route-specific quote. The merchant's obligation stays in GHS.
- A GHS price shown in the UI is not proof that the actual payment rail settles GHS. Do not label an order paid or merchant payable merely because the quote was accepted, a debit was requested, or a provider operation is pending.
- Model order/pricing denomination, customer source asset, merchant receivable and final payout asset as separate fields/concepts. Reconcile refunds against the captured transaction contract and actual settled currency.
- A merchant may explicitly opt into another denomination/settlement method only after its legal, accounting, payout and provider contract is defined; never infer it from the customer's asset.

### D7 — Savings and vaults separate target progress, actual assets, and locked obligations

- A savings goal can be only a target/progress tracker. It must not be shown as secured cash unless a real funded balance or an enforceable product obligation backs it.
- The goal/vault's denomination is pinned when created. The default for a domestic cedi goal is GHS; a deliberately USDC-denominated vault is a different contract.
- Keep actual funded asset, target denomination, committed/locked amount, available withdrawal amount and translated display estimate separate.
- For a GHS-denominated target backed by USDC, an estimated GHS progress amount remains approximate until the required GHS is actually funded or the precise product terms explicitly assign the currency risk. A nominal “GHS 5,000 target” does not turn a changing USDC holding into guaranteed GHS 5,000.
- Auto-save/round-up/scheduled deposits must go through the same funding/quote, consent, bounds, reservation, ledger and reconciliation rules as manual operations. A worker may not quietly bypass the user's currency contract.

### D8 — Wallet display is a valuation; spendability is asset-specific

- Show native GHS and real USDC holdings distinctly. A combined headline in GHS is labelled as an estimated/valued total when it includes USDC conversion; it must not be described as native GHS or guaranteed GHS proceeds.
- Keep available, reserved, escrowed, disputed, vault-locked, Susu-committed, pending and non-cash reward buckets separate, each with its native denomination.
- Never add Susu contributions to available balance. Susu commitment is a separate contract/locked bucket, calculated using the authoritative product contract and paid/settled contributions, not a recurring contribution amount, wallet valuation, or a guessed conversion.
- If FX is unavailable and USDC is held, do not show a fabricated complete combined GHS total or GH₵0.00. Show the native values independently and the combined value as unavailable/unknown.
- If the available API explicitly says valuation is unavailable or its contract is malformed, the frontend must fail closed to unknown/failed state. Client-calculated FX must not overwrite an explicit backend failure. A missing old-contract field may use only an explicitly documented temporary legacy display path, clearly labelled and removed when the contract is available.
- Build one reusable accessible cross-currency estimate component (including “≈”, clear “estimate” wording, rate basis/as-of when useful, and unavailable/stale state) for wallet, Susu, savings, vault, checkout and transfer screens. The number must account for known fees when it purports to estimate recipient proceeds. Never imply that a display estimate is an executable quote.

### D9 — Asset custody, omnibus pooling and provider sourcing are different choices

Do not assume “all users' USDC is in a pool, so the balance is theirs” without evidence and legal/operational controls.

A pooled/omnibus custody model can be considered only where permitted and supported by explicit custodian/provider arrangements, immutable per-user liability accounting, asset/network segregation, access controls, reconciliation and a documented insolvency/incident model. At every point, prove that:
- customer USDC liabilities by canonical asset/network are reconciled against eligible verified custody/provider assets and correctly identified in-transit items;
- customer assets, restricted assets and AZAMAN-owned treasury/inventory/revenue are distinguishable;
- pending/unverified deposits and ambiguous withdrawals are not silently counted as verified reserves;
- every custody movement has durable execution identity and provider/chain evidence;
- unsupported networks/assets are refused before mutation.

**Default new conversion strategy:** prefer quote-backed, transaction-specific provider sourcing when the product needs to acquire a new asset, rather than discretionary speculative house inventory. Where the provider has a supported direct-to-recipient route, consider direct delivery after validating the commercial contract, recipient/KYC requirements, quote/expiry semantics, webhooks and reconciliation obligations.

This does not mean “buy USDC from Kotani for every payment”:
- if the customer already holds USDC and is sending USDC, use their available, backed USDC and the appropriate transfer route;
- if the customer has USDC and a recipient must receive GHS, obtain a supported cash-out/conversion quote funded by the customer's reserved source assets;
- if a customer wants to acquire USDC using GHS, use an approved quote-in/acquisition route and only credit/execute after actual provider/custody settlement is proven;
- if a platform liquidity buffer is later needed for reliability, treat it as separately owned treasury inventory with bounded exposure, cost basis, immutable lots, inventory reconciliation and a separately approved treasury policy.

Kotani's public documentation describes quote-in/quote-out APIs and a Stellar USDC payout path to Ghana/Kenya. The public docs alone do not prove the current AZAMAN commercial contract supports every acquisition, conversion, direct-custody or delivery flow. Obtain current written provider confirmation before specifying a route. Moolre's current contracted fiat responsibilities remain as documented in the existing rail contract unless explicitly changed through an approved provider/architecture decision.

### D10 — Revenue is explicit and recognized at the correct time

- Do not continuously skim an embedded margin from customer balances or an omnibus pool. Do not use wallet FX revaluation to generate platform revenue.
- Earn conversion margin/fees at the specific quoted and settled transaction, with the economics represented in the authoritative ledger when realized.
- The customer quote must show the effective rate and all relevant costs. Where provider fee and platform margin can be independently evidenced, itemize provider fee, network charge, AZAMAN fee/margin, source debit and net recipient proceeds. If a provider only supplies an indivisible all-in price, disclose that all-in rate and only itemize components whose values are actually known; do not fabricate a provider/platform split.
- Revenue ownership, currency, realization point, refunds/reversals and tax/accounting treatment require explicit policy. A quote, an unexecuted order or an unrealized asset revaluation is not realized profit.
- Same-asset internal transfers require no FX spread; any optional transfer fee must be an explicit commercial rule and shown before confirmation. Network/provider costs for external withdrawals are disclosed and bound to the actual route.

### D11 — FX authorization gates are route-specific, not a blanket proxy for all money safety

A route that depends on converting/pricing in GHS must fail closed when a valid executable quote cannot be obtained or has expired. A display-only saved/demo rate never opens that route.

However, a pure same-asset USDC internal transfer does not mathematically require a USDC/GHS quote if its amount, fees, custody, ledger debit/credit, authorization and compliance checks are all asset-native. It should not automatically be blocked merely because the unrelated GHS display rate is stale. Similarly, a GHS-only transaction need not depend on USDC/GHS FX when a real native GHS asset/route fully funds it.

This is a route-by-route requirement, not a blanket allow-list:
- conversion, foreign-exchange pricing, GHS target proceeds, currency-valued limits or risk rules that use FX must have the required rate/quote authority;
- pure asset-native operations use their own authorization, balance, policy, network, sanctions/compliance, fee and settlement gates;
- if a particular risk limit genuinely depends on a combined GHS valuation, that limit must fail closed or use an approved conservative method when FX is unavailable.

**Implementation consequence:** backend PR #368's currently broad universal USDC/GHS freshness gate conflicts with this target for routes that do not depend on FX. Do not merge that policy as a permanent architecture default without narrowing/justifying each route and revising its route inventory/tests. Preserve all route-specific authorization and financial-integrity checks; this is not permission to bypass checks or enable unavailable routes.

### D12 — Susu payout reliability is a social and financial safety boundary

A Susu cycle can settle only when the cycle's required funding is final and reconciled according to its versioned contract, existing default/seizure rules and explicit net-pot calculation.

- Do not pay a guaranteed full GHS pot from an unconverted USDC valuation.
- A provider request accepted is not automatically a settled contribution. A timeout/ambiguous outcome must be put into a durable pending/reconciliation state; do not mark the member defaulted or penalize them solely because an external result is unknown.
- Do not pay out twice. One cycle payout identity/terminal claim must guard all payout, recipient credit, currency conversion, fee deduction, history and state changes.
- If collection/conversion/payout is delayed or ambiguous, hold the cycle in an explicit recovery state, notify affected participants and reconcile against provider evidence. The product must define maximum wait/grace and when it may transition to a policy-backed default.
- Default policy is **no platform advance and no partial payout**. Either requires a separate approved, funded risk/underwriting product with explicit loss limits, balance-sheet treatment, customer disclosure and tests. Never invent this guarantee in code.
- After the provider may have executed an irreversible payout, do not automatically refund on a timeout or retry with a new identity. Reconcile the original operation; only a definitive failure can use the documented reversal path.

### D13 — Legacy records are grandfathered, not reinterpreted

Before any migration, inventory current and legacy records by schema version, product type, contract version/hash, currency/asset semantics, amount field and historical ledger postings.

- Legacy rows with USDC-labelled contribution or vault amounts stay USDC-denominated unless the accepted contract and historical evidence prove otherwise.
- New GHS product contracts use explicit, immutable denomination fields and contract snapshots. Do not reuse or rename an amountUsdc field to store GHS.
- If a legacy group has ambiguous terms, preserve it in its verified historical state and define a safe closeout/re-contract process; no silent conversion of principal or promise.
- Schema change follows expand → dual-read/dual-write only where precisely justified → backfill under an auditable plan → switch authority → verify reconciliation → contract later. No destructive data rewrite or production backfill is part of this documentation decision.

## 5. Target end-to-end examples

### Example A — GHS Susu contribution funded by a mixed wallet

A member owes GHS 100.00. Their spendable GHS balance is GHS 70.00 and they hold some USDC.

1. The group contract and obligation remain GHS 100.00.
2. The planner identifies a GHS 30.00 target shortfall; it does not add GHS 70 to “USDC × rate” and treat that display number as cash.
3. The system obtains a current executable quote for the USDC source debit required to deliver the GHS 30.00 after known costs, and shows all-in terms and expiry.
4. After acceptance, GHS and USDC source amounts are reserved with durable identities. The conversion provider and ledger run through the established asynchronous settlement model.
5. The contribution is recognized only when the target GHS amount is secured under the route's settlement-finality contract. On a definitive failure, release/refund exactly once. On an ambiguous result, keep a recoverable pending state and do not penalize the member as a defaulter.
6. The group's pot and recipient payout are calculated only in the contractual GHS denomination, including explicitly disclosed group/product fees.

If GHS-native wallet funding/settlement is not yet live and reconciled for the applicable path, this journey is design-only and must not be enabled by client-side simulation.

### Example B — Diaspora member funds a GHS Susu for family in Ghana

The member may start with USDC. The group still promises a fixed GHS contribution and payout. At contribution time, the member can authorize a transaction-specific conversion to fund the agreed GHS obligation. The USDC amount required can change across future contribution dates, and the fees/rate are disclosed each time unless the member enabled a bounded recurring instruction.

The group records the GHS contribution after the defined settlement evidence. It does not keep a vague USDC equivalent in the pool and later assume it will cover the GHS payout.

### Example C — Explicitly USDC-denominated Susu

The contract says the contribution and payout are exact USDC amounts. The UI can show a GHS equivalent with ≈, rate basis and time. That estimate stays an estimate for the entire cycle; it does not become a fixed GHS promise because 60% of members hold or prefer a particular currency.

### Example D — GHS 100 marketplace or restaurant order paid from USDC

The order price snapshot and merchant receivable are GHS 100.00 if that is the merchant contract. The customer receives an explicit USDC-to-GHS quote showing source debit, costs, net target amount and expiry. The order becomes paid only on the canonical settlement outcome. Refunds follow the original captured contract and actual settled amounts, not today's FX display rate.

### Example E — “Send 50 USDC” vs “recipient receives GHS 100”

These are different transaction intents. In the first, the recipient receives the specified supported USDC asset amount and no FX conversion occurs unless explicitly chosen. In the second, the sender authorizes a GHS 100 obligation and the planner quotes the exact funding route. Recipient currency preferences can preselect the latter but cannot silently mutate the former.

## 6. Target system architecture and contract boundaries

### Core services (conceptual responsibilities, not permission to create duplicate services)

1. **Asset/Currency identity registry** — canonical currency codes and asset identifiers, network/contract/issuer identity, precision and eligible rails. Extend the existing registry/ledger metadata where possible.
2. **Product money contract** — immutable denomination, amount/fee/recipient/payout rules and accepted contract version for Susu, savings/vault and commerce.
3. **Asset-native balance read model** — authoritative exact asset-specific available/reserved/locked/pending buckets; display valuations are separate projections.
4. **Funding Planner** — selects from authorized spendable assets and eligible routes against a specific obligation; returns a plan, not proof of execution.
5. **Quote authority** — reuses current route-bound TransactionQuote/provider quote capability; pins amounts, route, fees, expiry and consent.
6. **Execution/orchestration boundary** — reuses durable FinancialOperation/idempotency and provider execution primitives; handles reservation, asynchronous provider results and recoverable unknown outcomes.
7. **Ledger** — the sole accounting authority, using the existing transaction-scoped ledgerService and explicit ASSET_CONVERSION semantics; no second ledger.
8. **Domain state machines** — Susu cycle, vault, savings, order and payment state change only under their own atomic transitions plus the canonical financial transaction/economic identity.
9. **Reconciliation** — reconciles customer liabilities by asset, custody/treasury/provider assets, in-transit items, GHS liquidity, inventory lots, restricted obligations, provider settlement and revenue.
10. **Frontend presentation and controls** — shows the server's contract, quote, asset buckets and state; gathers explicit choices/consent and reconciles to server outcomes. It never authorizes by calculating FX locally.

### Ledger/conversion posting requirements

- Each posting line has a canonical account and asset identity; ordinary posting groups must not mix incompatible assets.
- A cross-asset exchange is one explicit ASSET_CONVERSION economic identity with provenance linking the source and destination legs. Each asset ledger balances independently according to the established ledger contract; GHS and USDC quantities are never added together to make a balanced journal.
- The event records the amounts on each side, rate/quote and route evidence, fees/margin, and resulting balances. Exact conversion provenance and unique replay identity prevent duplicate conversion.
- GHS assets, USDC assets, customer liabilities, clearing, restricted obligations, inventory, treasury and revenue remain distinguishable. Do not count a customer liability, a synthetic pool, an unverified movement, or a clearing balance as verified reserve assets.
- User balance columns remain projections/compatibility fields during migration. A migrated path commits ledger postings, affected projections, domain state, transaction history, idempotency identity and durable audit/recovery data under one database transaction wherever the existing architecture permits. External calls are handled via durable orchestration/saga state rather than pretending they are atomic with PostgreSQL.
- Define and test currency-specific quantization at each provider/rail boundary. Keep internal exact precision; do not apply undocumented rounding, silent clipping or float conversion.

## 7. Cross-repository implementation plan

Implementation is staged by authority dependency, not by screen priority.

### Phase 0 — Contract and current-state audit (P0, before new money behavior)

**Repositories:** Planning + backend; read-only frontend/portal traces.

- [ ] Create the complete money-route matrix: every endpoint/worker for deposits, GHS/USDC conversion, internal transfers, crypto transfer, fiat withdrawal, Susu contribution/payout/default/refund, vault/savings deposit/release/auto-rule, marketplace/restaurant/dine-in checkout and refund, and provider callbacks.
- [ ] For each route, record source asset/currency, product obligation currency, recipient currency, quote authority, ledger postings, authorization, idempotency claim, provider state machine, settlement evidence, reversal and reconciliation.
- [ ] Verify P5-E/current Model B source and cost-basis realization state on backend main. Reuse the existing P5 accounting/quote/inventory boundary; do not duplicate it.
- [ ] Verify the native-GHS customer liability account, actual GHS deposit/funding path, spend/debit path, withdrawal path and provider reconciliation separately from the existing P5-D GHS liquidity/off-ramp authority.
- [ ] Map every currently USDC-labelled Susu/vault/savings field and the contractVersion/null legacy strata. Determine which current products are explicitly USDC-denominated versus merely storing contributions in USDC.
- [ ] Audit the full #368 route inventory and classify which routes require FX versus exact asset-native amount only. Make the gate contract conform to D11.
- [ ] Obtain current written Kotani/Moolre/custody-provider capability confirmation for quote-in, quote-out, direct destination delivery, supported assets/networks, rate locking, fee breakdown, settlement finality, timeout/status queries and responsibility for funds in transit.
- [ ] Obtain qualified Ghanaian legal/compliance advice on the intended custody, conversion, pooled funds, remittance, merchant settlement, Susu pooling, disclosures, advertising and corridor model.

**Exit:** documented route matrix, source-backed contract decisions, legal/provider blockers and no unresolved claim that a display value is spendable cash.

### Phase 1 — Freeze producer/consumer API contracts

**Repositories:** Planning, backend, frontend; Business/Admin where money/finance screens consume them.

- [ ] Define typed amount, currency code, canonical asset identity, source/target, exact/display projection, balance bucket, quote and settlement-state contracts.
- [ ] Pin product denomination and contract-version data for new product instances. Keep legacy interpretation explicit.
- [ ] Reuse current TransactionQuote, route identity, idempotency/FinancialOperation and ledger primitives. Add only missing fields/state required by audited paths.
- [ ] Define a server API for balance components and valuation status that distinguishes native holdings from estimates and explicit backend-unavailable state.
- [ ] Define Funding Planner request/response and quote-acceptance contracts before frontend integration. Do not invent endpoints just to fit a screen.
- [ ] Add cross-repo contract tests for response shapes, missing/null/malformed values, units, enums and forward-compatible additive fields.

**Exit:** one versioned API contract and a consumer compatibility matrix; backend remains the financial authority.

### Phase 2 — Complete the asset/settlement backbone

**Repository:** backend (main focus).

- [ ] Complete the code-level proof for real GHS customer liabilities and supported funding/debit/withdrawal/reversal paths; do not equate the existence of GHS ledger accounts or provider fiat liquidity with a funded user wallet.
- [ ] Prove how a GHS wallet is funded and backed, and how user GHS liability, custody/provider assets and GHS liquidity reconcile; define states for pending/unverified/external-in-transit amounts.
- [ ] Finish/review the explicit conversion and inventory economics authority against existing P5-A through P5-E work. A conversion cannot create unsupported customer USDC or GHS liabilities.
- [ ] Apply the target quote and route-specific FX gate policy. Pure asset-native operations should not depend on an unrelated FX observation; every path that needs a cross-currency price must require a valid executable quote.
- [ ] Add/extend provider adapters only after route contracts are verified. Confirm Moolre versus Kotani responsibilities; do not create duplicate fiat provider abstractions.
- [ ] Ensure authorization, quote acceptance, asset reservations, ledger posting, durable operation state, webhook processing and reconciliation handle retries, concurrent requests and ambiguous provider outcomes.

**Exit:** real-PostgreSQL proofs of cross-asset conservation, liability backing, settlement/reversal and provider ambiguity; reconciliation remains consistent; no live route is enabled by scaffolding alone.

### Phase 3 — Shared Funding Planner and quote execution

**Repository:** backend, then frontend contract integration.

- [ ] Plan obligations in their fixed denomination against exact spendable source-asset buckets.
- [ ] Reserve all required source amounts with a durable identity before external execution; prohibit duplicate spend across simultaneous requests.
- [ ] Implement the recoverable provider saga/operation lifecycle, with definitive failure, pending, ambiguous, success and reversal semantics. No automatic retry may create a second provider payment.
- [ ] Enforce one-use quote acceptance, expiry, recipient/route binding, amount binding, fees and target proceeds.
- [ ] Add bounded standing instructions for scheduled conversions; missing caps must fail closed.
- [ ] Expose canonical operation status/recovery API from existing patterns so frontend can continue pending operations without duplicating money movement.

**Exit:** database-backed race/replay tests, provider timeout-after-success recovery, quote expiry/conflict tests, and same-asset vs cross-asset behavior proven.

### Phase 4 — Susu product contract and cycle execution

**Repository:** backend first, frontend after API contract.

- [ ] Add explicit denomination and accepted version/hash for new groups; default new domestic groups to GHS only when the real GHS settlement path is proven.
- [ ] Keep existing legacy USDC group terms unchanged; classify or safely close out ambiguous groups through an audited, consent-based process.
- [ ] Separate member contribution obligation, source asset debited, quote/conversion, settled group-denomination contribution, pooled net amount, fee, winner payout denomination and destination settlement asset.
- [ ] Keep the group denomination immutable once activated/funded.
- [ ] Apply D12 pending/provider ambiguity and cycle payout contingency rules; don't punish a member for an unresolved provider status and don't front money without a separately approved funded risk program.
- [ ] Test the one-winner payout path, paid contribution commitment projection, defaults/seizures, grace state, full-pot mathematics, fee accounting, payout retries, provider failure and reconciliation.
- [ ] Keep cross-border USDC-denominated Susu as a separate explicit contract. Convert into a GHS group at contribution time when the group promise is GHS; do not delay conversion while silently keeping the group exposed to USDC/GHS drift.

**Exit:** exact contract-versioned cycle obligations and fully tested no-duplicate/no-unbacked-payout behavior.

### Phase 5 — Savings and vault contracts

**Repository:** backend first, frontend after contract.

- [ ] Define goal-tracking versus funded/locked savings as separate states.
- [ ] Pin denomination for each new goal/vault; preserve legacy USDC commitments and contract history.
- [ ] Make deposits, scheduled rules, round-up, early break, completion, penalty, release, and refund use the Funding Planner and exact product terms.
- [ ] Do not represent an unconverted USDC holding as a guaranteed GHS vault balance or guaranteed GHS proceeds.
- [ ] Fix projection/valuation filters against actual product state. Only contractually committed and actually funded amounts enter committed/locked buckets; do not sum configuration contribution amounts or completed/cancelled targets as money held.
- [ ] Test race of scheduled and manual deposit, early break vs maturity, quote/FX failure, provider uncertainty, idempotent release and currency precision.

**Exit:** locked liability matches the product contract and ledger; goal-only tracking is never represented as secured funds.

### Phase 6 — Local marketplace, restaurant and dine-in settlement

**Repositories:** backend, frontend, Business Portal and Admin Portal where required.

- [ ] Pin GHS price/order/merchant-receivable snapshots for domestic purchases.
- [ ] Allow an eligible USDC source only through a valid funding/settlement route that actually delivers the contracted merchant proceeds.
- [ ] Separate quote accepted, source reserved/debited, provider pending, merchant payable, payout pending and settled states.
- [ ] Reconcile cancellation/refund against the original order quote, captured source asset, actual delivered amount and explicit fee policy.
- [ ] Ensure merchant/admin portals read the same authoritative settlement/finance contract; no client-side pricing or success inference.
- [ ] Test inventory/payment races, double-confirm, source deduction vs payment pending, provider timeout, refund after rate changes and merchant payout reconciliation.

**Exit:** domestic checkout has a defensible GHS obligation and real matching settlement route; no “paid” UI without authoritative outcome.

### Phase 7 — Peer, recipient and cross-border currency semantics

**Repositories:** backend + frontend; other portals as appropriate.

- [ ] Add explicit “send asset amount” versus “recipient receives currency amount” intents.
- [ ] Model recipient preferred settlement currency as a preference, not an implicit conversion authorization.
- [ ] Define sender/recipient fee allocation before confirmation and preserve the accepted terms.
- [ ] Add future currency/asset support through canonical registries/route contracts, not hardcoded GHS/USDC branches.
- [ ] Keep internal same-asset movements distinct from external chain transfers and fiat off-ramps.
- [ ] Test internal no-FX transfer when FX display is unavailable, cross-currency quote expiry, wrong network/asset, recipient preference changes mid-flight and duplicate external delivery.

**Exit:** the receipt makes source debit, recipient amount/currency, fees and final settlement unambiguous.

### Phase 8 — Frontend and portal convergence

**Repository:** AZM-frontend first; Business/Admin Portal when money-facing contracts require it.

- [ ] Use shared typed models and shared estimate treatment; no client-side transaction price or authoritative FX arithmetic.
- [ ] Wallet headline shows valued estimate only when possible and clearly distinguishes native GHS, real USDC and restricted buckets.
- [ ] Funding screens preserve a visible source-asset/currency choice; ask for conversion approval when there is a shortfall or materially different transaction intent.
- [ ] Quote confirmation displays source amount, target amount, provider/route, fees/margin, expiry and what the recipient will receive.
- [ ] Pending operations have “continue/check status” recovery tied to the same operation identity; no blind retry/new idempotency key and no premature success animation/notification.
- [ ] Shared estimate component handles ≈, fee-aware projections, rate basis/as-of, unavailable state, screen-reader language, light/dark, 320/360/390dp, large text, reduced motion and accessibility target sizes.
- [ ] Client cannot silently replace explicit backend valuation failure with local fallback. Demo/display fallback can never authorize a transaction.

**Exit:** exact-head CI, API contract tests, visual/accessibility checks and end-to-end pending/recovery tests on the supported devices.

### Phase 9 — Reconciliation, governance and production gate

- [ ] Extend reconciliation by asset/network, user liability, reserved/locked amount, verified custody/provider balances, clearing/in-transit, GHS liquidity, inventory cost basis, realized fees and conversion amounts.
- [ ] Alert and stop affected routes on material liability/backing discrepancies, duplicate settlement, unclassified asset, stale quote authority or unreconciled external status.
- [ ] Produce an operator playbook for stuck operations, Susu cycle blockage, provider mismatch, FX outage, custody freeze, refund/reversal and customer communication.
- [ ] Set treasury exposure and any optional inventory limits through a separate approved policy; no implied house guarantees.
- [ ] Obtain legal/compliance/provider approvals for relevant routes and customer disclosures before live activation.
- [ ] Prove staging, migrations, rollback, backup/restore, observability, support playbooks, access controls and independent review.
- [ ] Run adversarial concurrency/replay tests and full end-to-end settlement rehearsal on safe, disposable/test environments before any separately authorized production activation.

## 8. Required test invariants and acceptance matrix

Every relevant test must assert financial rows and ledger/projection totals, not merely HTTP status or UI copy.

| Scenario | Required outcome |
|---|---|
| FX rate changes while customer holds 50 USDC | Actual USDC holding and liability remain 50 USDC; only a clearly labelled display valuation changes |
| GHS 100 obligation, GHS 70 available, USDC source offered | Planner identifies a GHS 30 shortfall; no contribution/payment until the quoted, approved route delivers/settles the remaining GHS |
| Quote expiry or changed recipient/amount/route | Refuse without executing under stale terms; require a new quote and approval |
| Provider unavailable before quote | No conversion, no fake success, no poisoned idempotency claim unless the established operation contract specifically requires a durable terminal attempt |
| Provider timeout after it may have executed | Keep same operation pending/reconciliation-required; no new provider submission or automatic refund without definitive outcome |
| Simultaneous spend of one source balance | Database-enforced winner(s) cannot collectively exceed spendable funds; exact losers have no financial residue |
| Cross-asset ledger posting without conversion identity | Reject before commit |
| Valid conversion | Exact source and destination legs link to one economic identity; each asset balances independently; fees/revenue reconcile |
| FX unavailable, user sends USDC to another user as USDC | May proceed only if source amount, fees, authorization, compliance, custody and same-asset ledger route are valid; GHS estimate shows unavailable, never GH₵0.00 |
| FX unavailable, user asks recipient to receive GHS from USDC | Fail closed for execution; no fallback/display rate is used as price |
| GHS-only action with real native GHS assets | Does not depend on unrelated USDC/GHS rate unless its own policy/limit requires it |
| Susu member contribution ambiguous at provider | Remains pending/reconciliation-required; is not counted as paid or a default; no penalty/payout based on an unknown result |
| Susu payout replay/concurrent calls | One committed payout and one terminal winner; no double wallet credit or duplicate provider transfer |
| Susu contract currency/fee changes after activation | Rejected; accepted terms and historical amount semantics remain unchanged |
| Mixed-member group has 50/50 or 60% preferred asset mix | No automatic currency switch; only pre-activation recommendation may use declared preferences |
| Savings goal without funded asset | May show goal progress only; cannot be reported as money held or locked |
| USDC vault with GHS display | USDC units remain exact; translated GHS value is marked ≈ and not a guarantee |
| Checkout refund after FX movement | Refund follows original contract/actual settled amounts and recorded fee policy, not current display FX |
| Platform fee/spread | Visible before consent; recognized only under settled ledger rules; no continuous customer-pool skim |
| Recipient's preferred currency changes after quote | Accepted operation preserves its recorded recipient currency and terms |
| Unsupported network/token (e.g. bridged substitute) | Rejected before financial mutation; asset identity cannot be inferred from a ticker alone |

Tests require real PostgreSQL for transaction, uniqueness, claim, ledger and concurrency properties, using a disposable test database—not the Neon production database. Exact-head CI and full diff audits remain required. Mocked-provider tests do not prove live provider behavior; live/testnet activity requires its own safety approval and evidence.

## 9. Open external decisions and hard blockers

The target product/accounting decisions above are settled recommendations. These items cannot be resolved by code inference and remain mandatory gates:

1. **Provider support:** current signed provider route, who holds the assets, who is counterparty, exact quote guarantees, limits, fees, settlement finality, webhook/status evidence, direct-to-recipient support and responsibility during transit. Public API docs are not a commercial contract.
2. **Legal/compliance:** counsel/regulator-confirmed permissions for custody, USDC conversion, pooled customer assets, Susu collection/payout, remittance, merchant proceeds, fees/spread, consumer disclosures, and each supported corridor.
3. **Legacy contracts:** exact meaning and current disposition of every legacy USDC Susu/vault record and current amount/payout semantics. Do not silently migrate ambiguous history.
4. **Susu risk policy:** confirm existing contractual default/seizure/grace rules against the intended GHS-denominated promise; any platform advance/partial payout is rejected by default and requires a separate funded policy.
5. **Commercial pricing:** decide fee schedules and revenue splits with finance/legal; architecture requires explicit quote disclosure and settlement-timed realization, but does not fabricate a particular fee percentage.
6. **Standing instruction limits:** per-product maxima for source debit, minimum target proceeds, maximum fees and permitted quote deviation must be approved; no global 2% default is assumed.
7. **GHS wallet activation:** prove native GHS customer liability funding, spend, withdrawal and reconciliation end-to-end. The existence of the native-GHS PR, GHS provider liquidity, ledger account, quote or display field is insufficient.
8. **FX gate:** revise/review PR #368's universal gate against D11 and a complete route inventory before it is treated as final policy.
9. **Supported asset set:** explicitly choose current supported asset/network and recipient settlement routes. Future USD/NGN and other tokens stay unavailable until that corridor's product, legal, provider and accounting contract is complete.

## 10. Regulatory and provider research (checked 2026-10-26)

- Bank of Ghana's [Virtual Assets page](https://www.bog.gov.gh/virtual-assets/) describes the Virtual Asset Service Providers Act, 2025 (Act 1154) as the legal foundation for registration/licensing and lists custodial/non-custodial wallet providers and fintech innovators using virtual assets among categories in scope.
- The Bank of Ghana and SEC's [joint public notice of 20 February 2026](https://www.bog.gov.gh/wp-content/uploads/2026/02/PRESS-RELEASE-PUBLIC-NOTICE-ON-UNAUTHORISED-ADVERTISING-OF-VIRTUAL-ASSET-AND-STABLECOIN-PRODUCTS-200226.pdf) directs VASPs, including sandbox participants, to refrain from mass marketing/public promotion of virtual-asset products unless expressly authorized. Treat this as an immediate marketing/legal review item; do not infer a licence or an exemption from an API/provider agreement.
- Kotani's [public API-v2 documentation](https://apispec.kotanipay.com/) describes quote-in/quote-out and a Stellar USDC payout route for Ghana/Kenya. It does not establish that every such route is available under AZAMAN's current signed agreement or that the customer is directly custodied by Kotani.
- Circle's [USDC terms](https://www.circle.com/legal/usdc-terms) describe USDC as designed to maintain a value of 1 USD, and disclose eligibility/transaction restrictions, non-insurance and other risks. USDC is not GHS and does not guarantee a fixed amount of cedi proceeds.

This is architecture planning, not legal advice. Counsel must validate current law, regulatory status and the intended operating model directly with the relevant authorities and providers before activation.

## 11. Definition of done

This architecture is implemented only when:

- one authoritative route/field/asset map has been audited on current main;
- product contracts explicitly pin denomination and preserve historical terms;
- the Funding Planner, route-bound quote, operation lifecycle and ledger use existing canonical authority boundaries;
- the exact GHS/native-asset funding and settlement path is real and reconciled where GHS liabilities are promised;
- all product paths account for mixed-asset funding correctly without cross-unit sums or fictional balances;
- Susu collection/payout is single-winner, adequately funded, recoverable on ambiguity and aligned with an accepted denomination contract;
- fees/margin are disclosed and realized accurately;
- route-specific FX requirements are documented and tested;
- cross-repo API contracts, UX estimate semantics, pending recovery, audit/reconciliation, legal/provider gates and production rollback are verified;
- real-PostgreSQL invariants, exact-head CI, independent source/diff review and adversarial tests pass;
- no P0 discrepancy remains between the promised currency, asset backing, ledger postings, provider settlement and client-reported outcome.

Until then, affected GHS-denominated money journeys remain planned/blocked or use only the exact currently supported contract. A display estimate, CI pass, API response or UI animation is never financial settlement evidence.
