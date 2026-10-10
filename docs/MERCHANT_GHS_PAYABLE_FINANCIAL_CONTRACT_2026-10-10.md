# AZM Marketplace — Merchant GHS Payable Financial Contract

> **Investigation output for the §3.3.3 workstream** directed by the finalized Mix/GHS owner decisions (2026-10-10, planning head `07e56de`). Companion to `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md` §3.3 and `MARKETPLACE_MIX_REDESIGN_2026-10-10.md` §9.
> **Status: DRAFT FOR OWNER + FINANCIAL REVIEW.** Documentation only. No application source code changed; PR #332, #333, #147 untouched. Nothing here may be treated as approved until the owner signs off and the durability-suite amendment (§7) is pinned.
> Verified against backend `main` `052370518133c91bdfb119ff271782045194f180` on 2026-10-10, plus the in-repo financial authorities: `docs/CURRENCY_AND_SETTLEMENT_CONTRACT.md`, `docs/PAYOUT_PROVIDER_OWNERSHIP_CONTRACT.md`, `docs/p5d-ghs-liquidity-authority.md`, `docs/p5e-model-b-settlement.md`, `services/ledgerService.js` (fail-closed chart of accounts).

**Design direction (owner-decided):** GHS merchant payable accounting with supported merchant payouts is the preferred local-market design. Merchant proceeds must NOT be automatically converted to USDC without explicit owner approval and a separately reviewed financial contract. This document specifies that GHS-native path for **GHS-paid marketplace orders only**; the USDC escrow settlement path is unchanged (R2 of the pricing contract: `priceUsdc`/`amountUsdc` semantics preserved everywhere).

---

## 1. What exists today (evidence, all traced)

- **Unit of account:** `docs/CURRENCY_AND_SETTLEMENT_CONTRACT.md` — USDC is the authoritative financial and settlement unit; GHS is presentation/fiat payout. A GHS-native merchant payable **amends this invariant for the marketplace order path** (§2 below); the amendment is additive and explicitly scoped.
- **Merchant settlement today:** escrow release credits `user:{payeeId}:liability` in USDC (`services/escrowService.js`); the merchant is the business owner's user wallet. Platform fee posts to `AdminProfitLog {amountUsdc, source, relatedTxId}` — a USDC-only table (`prisma/schema.prisma`; no currency column).
- **There is no merchant payout path.** The business portal has no withdrawal surface (Payroll is employee payroll); merchants off-ramp through the customer withdrawal rail (`/api/finance/withdraw`, USDC-denominated, GHS payout via Moolre).
- **Payout rails:** Moolre disbursement primary, MTN failover, via `PaymentFailoverService` (`services/moolreDisbursementService.js`, `services/paymentFailoverService.js`). The r15 payout-provider-ownership contract (`docs/PAYOUT_PROVIDER_OWNERSHIP_CONTRACT.md`) defines fail-closed ownership recording, dispatch-evidence-before-ownership write order, exactly-once settle/reverse (`financeService.completeFiatWithdrawal` / `reverseFiatWithdrawal`), the Withdrawal reconciliation record created inside the reservation transaction, and the reconciliation parking matrix. **This machinery is the payout authority the merchant rail must reuse — not re-invent.**
- **GHS liquidity authority (§P.5-D):** append-only `FiatProviderEvent` evidence is the source of GHS custody truth; `SystemFiatPool` is a derived projection updated deterministically inside the authority's transaction boundary; GHS amounts are exact `Decimal(20,2)` pesewas; typed provider outcomes (`NOT_DISPATCHED`/`DEFINITIVE_REJECTION`/`UNKNOWN_OUTCOME`/`DUPLICATE_REFERENCE`) with R15-B same-reference retry semantics.
- **Double-entry ledger (§P.5-A, `services/ledgerService.js`):** single-asset postings per entry; GHS/USDC never numerically balanced across an entry; `ASSET_CONVERSION` is the only cross-asset mechanism (requires `{identity, rate, quoteReference}`); **fail-closed account catalog** — non-canonical accounts and unrecognized dynamic patterns are rejected; the catalog already contains `'fiat:momo:ghs'` (GHS ASSET) and `'revenue:fees'` (USDC REVENUE), but **no GHS revenue account and no merchant-payable dynamic pattern exist** — they must be added deliberately, exactly like the P5 entries ("catalog entries only, unused until their wave").
- **Customer GHS collection (deposits):** Moolre collections are a deposit rail into the platform's own account, quoting USDC (`transactionQuoteService`); settled GHS is recorded as platform liquidity (§P.5-D) and the customer's USDC wallet is credited. **No part of this is a merchant settlement.**

---

## 2. Scope amendment to the currency invariant

`CURRENCY_AND_SETTLEMENT_CONTRACT.md` remains authoritative for everything except one sentence of its scope: for **marketplace orders paid via the GHS collection rail (P6)**, merchant earnings exist as a **GHS-denominated payable**, and merchant payout settles in GHS. USDC remains the authoritative unit of account for every other flow (escrow, conversions, withdrawals, order book, deposits). Concretely:

1. **No conversion on this path, ever.** GHS-paid orders never create an `ASSET_CONVERSION` entry; no USDC value is synthesized for the merchant; no GHS value is derived from an oracle rate for accounting purposes. The GHS amounts are the listing-snapshot and collection-confirmed amounts, exactly.
2. **One path, one currency.** A given order's payment record dictates its settlement currency: GHS-paid → GHS payable; USDC escrow → existing USDC settlement. Mixed-currency settlement of one order is prohibited.
3. **The amendment merges as documentation with the P6 PR**, never before, and the durability suite must pin §7 before any settlement code lands.

---

## 3. Merchant payable accounting model (§3.3.3-a)

### 3.1 Account catalog additions (fail-closed catalog, deliberate entries)

```
// NEW — merchant payable sub-ledgers (GHS only; added to the catalog deliberately,
// unused until P6):
'merchant:{bizId}:payable:ghs'   accountClass LIABILITY  normalSide CREDIT asset GHS
'merchant:{bizId}:receivable:ghs' accountClass ASSET     normalSide DEBIT  asset GHS
'revenue:fees:ghs'               accountClass REVENUE    normalSide CREDIT asset GHS
'clearing:payout:pending:ghs'     accountClass CLEARING   normalSide DEBIT  asset GHS
```

- `merchant:{businessProfileId}:payable:ghs` — **business-scoped, not user-scoped.** Payable follows the KYB business identity (the entity that owns the storefront and prices), not the owner's personal wallet. This is the key difference from today's `user:{payeeId}:liability` escrow credit, and it is what makes "merchant" a first-class financial counterparty. A business may have multiple owner users; none of them "owns" the payable balance directly.
- The dynamic pattern must be registered alongside the existing `USER_LIABILITY_RE`-style patterns in the fail-closed catalog, with the same rejection semantics for malformed ids.
- `revenue:fees:ghs` is a separate account from `'revenue:fees'` (USDC): per-asset leg balance already forbids mixing, and reporting must aggregate fees per currency without post-hoc conversion.

### 3.2 Order-confirmation postings (single-asset GHS entries, exact pesewas)

When the P6 collection webhook confirms a GHS payment against an order (quote-matched, typed outcomes per §P.5-D), inside the provider-confirmation transaction:

```
gross   = settled GHS (provider-confirmed, quote-matched, exact 2dp)
fee     = commission (§5, computed from the listing snapshot, exact 2dp)
payable = gross - fee

D fiat:momo:ghs              gross        (platform GHS asset up, per §P.5-D evidence)
C merchant:{biz}:payable:ghs  payable      (merchant liability up)
C revenue:fees:ghs           fee          (platform GHS fee revenue)
```

- The posting references the order's **three-fact snapshot** (listing group, payment group, payable group) so the ledger line, the order row and the fee record are one auditable unit (`relatedTxId` = order payment reference; idempotent on the collection externalRef — replays converge on the existing row, never double-post).
- **Ordering guarantee:** postings happen only after the provider-confirmed state; `UNKNOWN_OUTCOME` never posts. A confirmation is idempotent exactly-once via the §P.5-D dedup key plus the ledger's economic-identity guard (`LEDGER_REPLAY_IN_CLAIMED_OPERATION` pattern).
- The customer GHS collection itself (money into the platform Moolre account) is recorded by the existing collection rails; this entry models the split of that settled GHS into merchant payable and platform fee. `fiat:momo:ghs` debits here must equal the collection rails' credited GHS — the §P.5-D authority reconciles the asset side.

### 3.3 Fee record (additive, USDC authority untouched)

`AdminProfitLog` stays USDC-only (`amountUsdc` is its authority; R2). NEW `MerchantFeeGhs` table (additive): `id`, `businessProfileId`, `orderId`, `amountGhs Decimal(20,2)`, `feeScheduleId`, `computedFrom` (the listing-snapshot line totals), `relatedTxId` (the ledger entry id), `createdAt`. Every GHS fee has an order reference and a ledger twin; reporting sums per currency and never converts.

### 3.4 Projections (derived, deterministic)

`MerchantPayableSummary` (per business): `payableGhs`, `pendingPayoutGhs`, `receivableGhs`, `lifetimeFeeGhs`, `lastSettledAt` — a derived projection reconciled nightly against the ledger, exactly the §P.5-D pattern (projection updated inside the authority's transaction boundary; never the source of truth). The business portal displays this summary; it never computes payable from order rows.

---

## 4. Merchant payout rail (§3.3.3-b)

**Supported rail at launch: Moolre MoMo disbursement to the merchant's registered MoMo number** (MTN failover per `PaymentFailoverService`), reusing the r15 machinery verbatim:

1. **Initiation:** merchant requests a payout (portal) for an amount ≤ `payableGhs` (the projection is advisory; the authoritative check is inside the transaction). The payable debit and payout reservation happen **in one transaction**: `D merchant:{biz}:payable:ghs` / `C clearing:payout:pending:ghs`, plus the Withdrawal-shaped reconciliation record created inside that same transaction (the r15 controller reservation contract: if the record cannot be established, the whole reservation rolls back — fail-closed, no provider I/O).
2. **Dispatch:** through the existing disbursement chain. After provider acceptance, the r15 write order applies unchanged: dispatch evidence → canonical ownership write → `IN_TRANSIT` transition; every failure path parks with a durable exception; the user (merchant) is never refunded after acceptance.
3. **Settlement:** provider-confirmed success → `D clearing:payout:pending:ghs` / `D fiat:momo:ghs` (asset down) / settle exactly once via the r15 terminal claim; late provider failure that arrives after settlement must not claw back (same rule as the fiat withdrawal lifecycle).
4. **Reversal:** provider-confirmed failure of a still-pending payout → the reservation reverses: `D clearing:payout:pending:ghs` / `C merchant:{biz}:payable:ghs` — the payable balance is restored exactly, idempotent re-reconcile.
5. **Reconciliation:** the existing `withdrawalReconciliationWorker` parking matrix applies verbatim (UNKNOWN ownership → exception queue check → park, never cross-rail guessing).
6. **Provider fees:** any Moolre disbursement fee is borne by the platform at launch (recorded as GHS cost against `revenue:fees:ghs`, with the provider fee visible in the payout record) — a decision point in §9; merchant-borne fees are NOT assumed.
7. **Bank-transfer payouts: explicitly unsupported until a bank rail is integrated and audited.** No speculative design; the contract names MoMo only because MoMo is the audited rail.

**Payout availability rule:** payable funds are eligible for payout only from the provider-confirmed state. There is no payout against pending/unsettled collections, and no advance/credit facility. Minimum payout threshold and schedule (on-demand vs batched) are §9 decision points.

---

## 5. Fee/commission model in GHS (§3.3.3-c)

- Commission is computed **at order-payment confirmation time from the GHS listing-snapshot line totals** — never from a converted USDC value, an oracle rate, or a payment record in another currency.
- All arithmetic in integer pesewas (BigInt); rounding mode round-half-up at 2dp; **fee ≤ gross** enforced structurally; fee floor/cap and the percentage live in a versioned `MerchantFeeSchedule` (config, not code), so a schedule change never rewrites historical fees.
- Whether the GHS schedule mirrors the escrow `feePct` (`services/escrowService.js` `_round6(amount * feePct)`) or carries marketplace-specific rates is a §9 decision point. The escrow path is untouched in any case.

## 6. Refund and dispute handling (§3.3.3-d)

- **Refund currency == payment currency** (already owner-decided). A GHS-paid order refunds in GHS to the customer's MoMo via the disbursement rail; no USDC is created or consumed on either side.
- **Payable-first refund ordering:** a refund first debits the merchant's still-settled payable (`D merchant:{biz}:payable:ghs`); if the payable is insufficient (earnings already paid out), the refund is funded from platform GHS liquidity and the merchant owes the platform: `D merchant:{biz}:receivable:ghs` (asset up — a real receivable, never a negative payable). Future payouts are blocked while a receivable exists for that business.
- Refund idempotency mirrors the collection path: exactly-once on the refund reference; the refund record carries the original payment group reference.
- **Disputes** use the existing parking/exception machinery (`ReconciliationException`) and the control-plane four-eyes force-action governance; no automatic claw-back of a settled payout ever occurs — reversal of a settled payout is an audited manual decision, mirroring "the user is NEVER refunded after acceptance" with the merchant symmetric ("the merchant is never debited after settlement except via an audited receivable decision").

## 7. Financial invariants amendment (§3.3.3-e) — durability-suite pins

The P6 PR must extend the financial-durability suite with these pins before any settlement code merges (each becomes a real PG test):

1. **Conservation per order:** `payable + fee == gross` for every confirmation posting, on exact pesewas.
2. **No cross-asset leakage:** no GHS-paid order path ever posts `ASSET_CONVERSION`; GHS and USDC entries never share a posting; the payable, receivable and fee accounts never hold USDC.
3. **Exactly-once:** duplicate collection webhooks, duplicate payout dispatches and re-reconciled reversals each converge to one economic mutation (dedup keys + terminal claims).
4. **No negative payable:** `merchant:{biz}:payable:ghs` never goes debit-balance; a refund beyond payable forces the receivable path, which is itself a durable row with four-eyes release.
5. **Snapshot-linkage integrity:** every confirmation posting references an order with its three fact groups intact; a payment record without its listing snapshot or fee twin fails reconciliation.
6. **Projection equality:** `MerchantPayableSummary` equals the ledger-derived balances per business (nightly proof, mismatch parks an exception — never silent drift).
7. **R15 ownership compliance:** merchant payouts inherit the ownership contract's tests (evidence-before-ownership write order, fail-closed identity, parking matrix, never-refund-after-acceptance).
8. **Fee invariants:** fee schedule version recorded per fee; fee ≤ gross; schedule changes never recompute historical fees.
9. **Refund invariants:** refund ≤ payment; refund currency == payment currency; settled-payout reversals only via audited manual paths.

## 8. Rollout gates (in order)

1. Owner sign-off of THIS document (including the §9 decision points).
2. Catalog + schema PR: account entries (catalog-only, unused), `MerchantFeeGhs`, `MerchantPayableSummary`, payable/receivable dynamic patterns, payout record wiring — zero behavior change, durability suite already pinning §7 where testable statically.
3. GHS collection rail (P6a) with order-linked references, behind its own review.
4. Settlement + payout PR (P6b): confirmation postings, payable projection, merchant payout endpoints reusing r15 — full §7 durability battery required.
5. Four-eyes monitoring on the first live merchant payouts; reconciliation-exception queue alerting; then automatic payouts only if the owner separately approves (§9).

## 9. Decision points for the owner (none may be assumed by implementation)

1. **Sign-off of this contract** — including the §2 scope amendment to `CURRENCY_AND_SETTLEMENT_CONTRACT.md`.
2. **Fee schedule:** mirror escrow `feePct`, or marketplace-specific GHS schedule (recommended: separate versioned schedule, so marketplace pricing can evolve independently).
3. **Payout fee bearer:** platform absorbs provider disbursement fees at launch (recommended) vs merchant-borne.
4. **Minimum payout threshold + schedule:** on-demand with a threshold (recommended), vs automatic periodic batch.
5. **Business-scoped payable:** confirm the KYB business is the financial counterparty (recommended) rather than the owner user's wallet.
6. **Payout destination:** merchant's registered business MoMo (recommended; same KYB identity) vs owner personal MoMo — a compliance-relevant choice.
7. **Automatic payout of unclaimed balances:** explicitly out of scope at launch; only on separate approval.

— End of merchant GHS payable investigation draft. Documentation only; no application source code changed.
