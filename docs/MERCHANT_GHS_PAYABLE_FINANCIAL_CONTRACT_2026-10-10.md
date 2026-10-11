# AZM Marketplace — Merchant GHS Payable Financial Contract

> **Revision 2.1 (2026-10-11) — verification pass over rev 2 (2026-10-10).** No accounting change: every §4 posting verified balanced per asset against the owner's correction list. Rev 2.1 adds the explicit **immutable internal business identity** definition (§6.1), the MerchantPayout correlation-record statement (§6.1), and labels the §3 account-catalog entries as **future implementation work**. Revision 1 (`b5b5ec3`) is superseded: the clearing-account normal side, the settlement posting, the provider-fee accounting, the refund postings, the P5-D integration, the payout record economics and the release gates have all been corrected per the owner's review. Supersedes revision 1 (`b5b5ec3`) and rev 2 (`80f9204`). Companion to `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md` §3.3 and `MARKETPLACE_MIX_REDESIGN_2026-10-10.md` §9.
> **Status: DRAFT FOR OWNER SIGN-OFF.** Documentation only. No application source code changed; PR #332, #333, #147 untouched; nothing merged. No GHS settlement code may be written until this contract is signed off AND the §8 release gates are met.
> Verified against backend `main` `0523705` on 2026-10-10: `services/ledgerService.js` (chart of accounts, `post` contract, per-asset balancing), `src/services/fiatLiquidityService.js` (receipts, evidence chains, reservations), `docs/p5d-ghs-liquidity-authority.md`, `docs/PAYOUT_PROVIDER_OWNERSHIP_CONTRACT.md` (r15), `services/finance.service.js` (`clearing:fiat:offramp:usdc` posting), `services/modelBSettlementService.js` (`fiat:momo:ghs` precedent), `services/moolreDisbursementService.js:269` (amount serialization boundary).

**Design direction (owner-decided, §9):** GHS-native, business-scoped merchant payables with supported merchant payouts; automatic conversion of merchant GHS proceeds to USDC is PROHIBITED unless a separately reviewed contract is explicitly approved. This contract specifies the GHS-native path for **GHS-paid marketplace orders only**; the USDC escrow settlement path and the entire existing USDC chart are unchanged.

---

## 1. What exists today (evidence, all traced)

- **Unit of account:** `docs/CURRENCY_AND_SETTLEMENT_CONTRACT.md` — USDC is authoritative; GHS is presentation/payout. §2 amends the scope for GHS-paid marketplace orders only.
- **Merchant settlement today:** escrow release credits `user:{payeeId}:liability` in USDC; platform fee posts to `AdminProfitLog {amountUsdc}` (USDC-only table). **No merchant payout path exists** — merchants off-ramp through the customer withdrawal rail.
- **Ledger authority (`services/ledgerService.js`):** fail-closed chart of accounts (sign, class, asset and normal side are part of each account's canonical identity; a persisted row that disagrees refuses to post); `post()` requires the CALLER-OWNED transaction, a durable idempotencyKey, ≥2 lines, exactly one non-zero side per line, and **per-asset balance — a normal posting lives in ONE asset; `ASSET_CONVERSION` is the only cross-asset mechanism and is never used on this path**; exact decimal strings (`toFixed(2)` for GHS); replay converges (`replayed=true`, no second posting).
- **Clearing precedent:** `clearing:fiat:offramp:usdc` is `CLEARING`/`CREDIT`-normal — "its positive balance is the OUTSTANDING value handed to the fiat rail awaiting GHS-side reconciliation"; credited when principal enters the rail, debited when the fiat asset movement is confirmed (§P.5). The merchant payout clearing accounts mirror exactly this semantics.
- **GHS asset precedent:** `fiat:momo:ghs` (`ASSET`, `DEBIT`-normal, GHS) is the catalogued platform MoMo GHS asset; Model B deposits land `D fiat:momo:ghs / C equity:treasury:ghs` from evidenced settlement. Order collections use the same asset account but credit merchant payable + fee revenue — merchant-owned GHS never enters treasury equity.
- **GHS-liquidity authority (§P.5-D, `src/services/fiatLiquidityService.js`):** append-only `FiatProviderEvent` raw evidence; `FiatLiquidityReceipt` (AVAILABLE created only by service-verified transitions — matched-deposit chain, reconciliation match, or treasury opening; never caller say-so; full semantic identity per dedupKey, §L quote-substitution closure); `FiatLiquidityReservation` (RESERVED → IN_TRANSIT → PAID_OUT | RELEASED | RECONCILIATION_REQUIRED, single-winner gte-CAS claim); `SystemFiatPool` a derived projection only.
- **Deposit receipts are deposit-specific:** `verifyDepositEvidenceChain` requires a `TransactionHistory` row that IS a `DEPOSIT_FIAT`, IS `COMPLETED`, has a consumed route-bound `TransactionQuote`, and a durable INBOUND provider event matching provider, amount and reference exactly. **An order-linked collection is none of these** — no USDC amount, no wallet deposit, no quote of that class.
- **Payout rails:** Moolre MoMo disbursement primary, MTN failover (`PaymentFailoverService`), under the r15 ownership contract: dispatch evidence → canonical ownership write → IN_TRANSIT, fail-closed identity, one terminal winner, durable exceptions, never refund after provider acceptance. **Moolre MoMo is the only proposed launch payout rail; no bank payout is claimed.**
- **Serialization defect (traced):** `services/moolreDisbursementService.js:269` serializes the transfer amount as `String(Number(amountGhs))` — a binary floating-point boundary. It must be corrected before P6b dispatch (§6.4); authoritative money must never pass through `Number()`.

---

## 2. Scope amendment to the currency invariant

`CURRENCY_AND_SETTLEMENT_CONTRACT.md` remains authoritative except for one scoped amendment: for **marketplace orders paid via the GHS collection rail**, merchant earnings exist as a business-scoped GHS payable and merchant payouts settle in GHS. USDC remains authoritative everywhere else.

1. **No conversion on this path, ever.** No `ASSET_CONVERSION` entry; no USDC synthesized for the merchant; no oracle-derived accounting amounts. **Automatic conversion of merchant GHS proceeds to USDC is PROHIBITED unless a separately reviewed contract is explicitly approved by the owner.**
2. **One order, one settlement currency.** The payment record's currency dictates settlement: GHS-paid → GHS payable; USDC escrow → existing USDC path unchanged. Mixed-currency settlement of one order is prohibited.
3. **USDC payment for a GHS-priced order (owner-decided):** requires a separate, purpose-specific, approved server quote (order-checkout purpose, explicit expiry, explicit customer confirmation) — the public trading oracle (`/api/oracle/rates`) is a presentation feed and is NEVER the checkout quote. That quote contract needs its own approval before any such payment path is built (§9).
4. The amendment merges as documentation with the P6 PR; §8 gates precede any settlement code.

---

## 3. Account catalog additions (fail-closed chart, deliberate entries)

All added to `CANONICAL_ACCOUNTS` (and the dynamic-pattern registry) as catalog-only entries, each with canonical sign, class, asset and normal side. **These are future implementation work:** nothing below is added to any codebase by this document; the entries land with the P6 catalog/schema PR (additive, no behavior), remain unusable until the settlement code that references them also lands, and are fail-closed by the chart's persisted-identity guard in the meantime. This revision changes no application source.

| Account | Class | Normal side | Asset | Meaning |
|---|---|---|---|---|
| `merchant:{businessProfileId}:payable:ghs` | LIABILITY | CREDIT | GHS | What the platform owes the KYB business for GHS-paid orders (business-scoped — never a user wallet). Positive balance = credit side. |
| `merchant:{businessProfileId}:receivable:ghs` | ASSET | DEBIT | GHS | What the business owes the platform (refund beyond already-paid earnings). |
| `revenue:fees:ghs` | REVENUE | CREDIT | GHS | Marketplace commission in GHS — separate from `revenue:fees` (USDC); per-asset balancing forbids mixing and reporting aggregates per currency without conversion. |
| `clearing:payout:pending:ghs` | CLEARING | **CREDIT** | GHS | **Corrected (rev 1 wrongly proposed DEBIT-normal).** Mirrors `clearing:fiat:offramp:usdc`: positive balance = outstanding GHS committed to the merchant payout rail awaiting provider-confirmed movement. CREDITed at reservation; DEBITed exactly once at provider-confirmed settlement or reversal. |
| `clearing:refund:pending:ghs` | CLEARING | CREDIT | GHS | Same semantics for customer-bound GHS refunds — a distinct counterparty never shares the merchant-payout clearing account. |
| `expense:provider:fees:ghs` | EXPENSE | DEBIT | GHS | **NEW (rev 1 wrongly booked provider costs against a revenue account).** Provider disbursement costs absorbed by the platform, recognized from provider evidence at settlement. |

New `JournalEntryType` enum values (additive): `MERCHANT_COLLECTION`, `MERCHANT_PAYOUT`, `MERCHANT_REFUND`. The existing USDC chart and every historical posting are untouched.

## 4. Balanced postings (the complete economic examples)

All amounts are exact 2dp GHS strings from the pesewas/Decimal authority (`toFixed(2)`); every posting is single-asset GHS, per-asset balanced, inside the caller-owned economic transaction, with a durable idempotencyKey; replays converge without a second posting. Notation: `biz` = the immutable internal business identity, `businessProfileId` (§6.1 — never a userId, wallet address or provider id); `X` = payout amount; `F` = provider fee; `R` = customer refund amount.

### 4.1 Order-linked collection confirmed (collection + fee recognition)

Provider callback confirms the GHS collection for order payment reference `ref`:

```
MERCHANT_COLLECTION  (idempotencyKey: ledger:merchant:collection:{ref})
D fiat:momo:ghs                gross
C merchant:{biz}:payable:ghs    payable = gross − fee
C revenue:fees:ghs              fee
```

Balanced: `gross = payable + fee` (conservation pinned). This is the **single point of GHS asset recognition** (§5.3). The `MerchantFeeGhs` row and the payable projection update commit in the same transaction (§5.4).

### 4.2 Payout reservation (merchant-initiated, manual only)

```
MERCHANT_PAYOUT  (idempotencyKey: ledger:merchant:payout:reserve:{payoutRef})
D merchant:{biz}:payable:ghs    X
C clearing:payout:pending:ghs   X
```

Balanced. The merchant's payable is released into the rail clearing account atomically with the liquidity reservation and the `MerchantPayout` record (§6.1) — the committed amount is immediately unavailable for another payout (single-winner reservation, §6.3).

### 4.3 Dispatch-related transitions — NO ledger posting

Provider acceptance, the r15 ownership writes (dispatch evidence → canonical ownership → IN_TRANSIT) and reconciliation parkings are **record-state transitions on `MerchantPayout` and `FiatLiquidityReservation`, never ledger postings**. The ledger moves only at reservation and terminal provider-confirmed outcomes. This matches the user-withdrawal rail, where deferred economics realize only at provider SUCCESS.

### 4.4 Provider-confirmed payout SUCCESS (settlement + provider cost)

```
MERCHANT_PAYOUT  (idempotencyKey: ledger:merchant:payout:settle:{payoutRef})
D clearing:payout:pending:ghs    X
D expense:provider:fees:ghs      F        (only if the provider charged a fee)
C fiat:momo:ghs                  X + F
```

Balanced: `X + F = X + F`. The GHS asset falls by exactly what left the platform account per provider evidence; the pending clearing balance clears exactly once; the platform absorbs `F` (owner-decided) as a GHS expense, **not** against revenue. When `F = 0` the posting is `D clearing X / C fiat X`. **The deduction model is rejected:** the merchant must receive exactly `X`; a provider configuration that delivers `X − F` to the merchant fails closed before dispatch.

### 4.5 Provider-confirmed payout FAILURE (reversal of a pending payout)

```
MERCHANT_PAYOUT  (idempotencyKey: ledger:merchant:payout:reverse:{payoutRef})
D clearing:payout:pending:ghs    X
C merchant:{biz}:payable:ghs     X
```

Balanced. Payable restored exactly; idempotent re-reconcile. A late failure callback after a terminal SUCCESS claim must not claw back — the same no-clawback rule as the fiat withdrawal lifecycle; contradictory evidence parks (`RECONCILIATION_REQUIRED`), never reverses.

### 4.6 Refund — payable sufficient

Commit (atomic with the refund record and its reference):

```
MERCHANT_REFUND  (idempotencyKey: ledger:merchant:refund:commit:{refundRef})
D merchant:{biz}:payable:ghs    payablePortion
D revenue:fees:ghs              feePortion          (commission reversal)
C clearing:refund:pending:ghs   R = payablePortion + feePortion
```

Balanced. Amounts derive **only** from the order's immutable listing/payment snapshots and the fee-schedule version recorded on the original `MerchantFeeGhs` row — never recomputed from a different schedule. Full refund: `payablePortion + feePortion = gross` (the entire commission reverses). Partial refund: the same line-level derivation for the refunded lines. **No hidden merchant charge:** the merchant's payable falls by exactly the refunded lines' payable portion, and the reversed commission is returned to it — an incomplete payable model must never leave the merchant net-negative on a refunded order.

Disbursement to the customer's MoMo confirmed:

```
MERCHANT_REFUND  (idempotencyKey: ledger:merchant:refund:settle:{refundRef})
D clearing:refund:pending:ghs    R
D expense:provider:fees:ghs     F       (platform-absorbed refund-dispatch fee, if any)
C fiat:momo:ghs                 R + F
```

Disbursement failure: reverse the commit exactly — `D clearing:refund:pending R / C payable payablePortion / C revenue:fees feePortion` (the customer refund obligation is cancelled, payable and commission restored).

### 4.7 Refund — payable insufficient (receivable path)

When the refunded lines' payable portion exceeds the business's remaining payable (earnings already paid out):

```
MERCHANT_REFUND
D merchant:{biz}:receivable:ghs   shortfall = payablePortion − payableAvailable
D merchant:{biz}:payable:ghs      payableAvailable
D revenue:fees:ghs                feePortion
C clearing:refund:pending:ghs     R
```

Balanced: `shortfall + payableAvailable + feePortion = payablePortion + feePortion = R`. The customer is made whole from platform GHS liquidity; the merchant owes the shortfall as a durable receivable. Payable **never goes negative**; a receivable blocks further payouts for that business until settled under a four-eyes audited release. Reversal of a settled payout is only ever an audited manual decision — never automatic.

Every posting above is a complete, balanced entry; no path leaves a dangling one-sided or cross-asset line, and no path posts a USDC amount.

---

## 5. Integration with the GHS-liquidity authority (§P.5-D)

### 5.1 A new receipt class — order-linked collections are NOT deposits

`recordReceipt`'s matched-deposit path requires `DEPOSIT_FIAT` + a consumed route-bound quote + a USDC wallet credit — a shape an order collection must never borrow: **no `TransactionHistory` USDC amount is manufactured, and no user wallet is credited.** P6 adds a fourth receipt path to the same authority — `recordOrderCollectionReceipt` — with its own fail-closed evidence chain, mirroring `verifyDepositEvidenceChain`:

- a durable INBOUND `FiatProviderEvent` exists, reports a successful collection, matches the receipt's provider and **amount exactly**, and its `relatedReference` equals the order payment reference;
- the order-linked **collection intent** (created at checkout: unique collection reference + exact quoted GHS amount for that order payment) exists and matches the event amount exactly — the receipt must name **its own** intent identity (§L-style substitution closure; a twin intent with identical economics must not unlock liquidity);
- the order's payment record exists, is the payment named by the intent, and is CAS-claimed `PENDING → COMPLETED` **inside the same caller transaction**;
- the receipt's dedupKey carries the economic class `ORDER_COLLECTION` — distinct from deposit dedupKeys, so one real collection can never be claimed as both.

Only then does the receipt land AVAILABLE. Amount mismatch, unknown intent, contradictory evidence → quarantine (`RECONCILIATION_REQUIRED`), never auto-credit.

### 5.2 Atomicity — one transaction per economic boundary

**Collection confirmation** (single caller-owned transaction): FiatProviderEvent (already durable, append-only) → receipt evidence verification → order payment CAS claim → ledger posting (§4.1) → `MerchantFeeGhs` row → payable projection update. Any failure rolls back the whole transaction; a replay converges on every element (receipt dedupKey, ledger idempotencyKey, CAS claim) with zero mutation.

**Payout reservation** (single transaction): `MerchantPayout` record creation → `reserveForPayout` single-winner liquidity claim (gte-CAS; a loser rolls back everything, including the ledger posting §4.2) → destination re-verification. **The reservation must cover the full evidenced asset outflow of the dispatch — principal PLUS platform-absorbed provider fee coverage per §6.5** (reserving only `X` understates the committed GHS by `F` and overstates the pool projection; §6.5 is an open blocker). The same coverage rule applies to the refund disbursement (`R + F`, §4.6). **Terminal outcomes** (single transaction, worker-driven): provider outcome typed per §P.5-D (`NOT_DISPATCHED`/`DEFINITIVE_REJECTION`/`UNKNOWN_OUTCOME`/`DUPLICATE_REFERENCE`) → terminal single-winner claim on `MerchantPayout` + `settleReservation`/`releaseReservation` → the §4.4/§4.5 ledger posting exactly once.

### 5.3 Where the GHS asset is recognized — exactly once

The **only** point of GHS asset recognition on this path is the `MERCHANT_COLLECTION` posting's `fiat:momo:ghs` debit (§4.1), sourced exclusively from durable provider evidence. Distinct responsibilities, one real collection:

- **Provider evidence** — append-only `FiatProviderEvent` (raw observation, never rewritten);
- **Liquidity availability** — the `FiatLiquidityReceipt` (ORDER_COLLECTION class) making settled GHS available for payouts; `SystemFiatPool` stays a derived projection updated deterministically inside the authority's boundary;
- **Ledger truth** — the single `fiat:momo:ghs` debit.

Reconciliation proves the three are 1:1:1 per collection reference: no receipt without its event, no ledger asset debit without its receipt, and no second debit for the same reference (the ledger's economic-identity guard plus the receipt dedupKey are the race fences). One real collection is never counted as multiple asset increases — the deposit rail, the treasury path and the order path have disjoint dedupKey classes and disjoint evidence chains.

### 5.4 Projections

`MerchantPayableSummary` (per business): `payableGhs`, `pendingPayoutGhs`, `receivableGhs`, `lifetimeFeeGhs`, `lastSettledAt` — derived only, updated inside the authority's transactions, reconciled nightly against the ledger (mismatch parks an exception, never silent drift). `MerchantFeeGhs` (additive table): order reference, fee-schedule version, computed-from snapshot line totals, ledger entry id. `AdminProfitLog` stays USDC-only.

---

## 6. Merchant payout rail — r15 safety, business-scoped economics

### 6.1 The payout record is not a user withdrawal

NEW `MerchantPayout` table (business-scoped): `businessProfileId`, `reference`, `amountGhs Decimal(20,2)`, `destinationId`, `status` (`PENDING → DISPATCHED → IN_TRANSIT → PAID_OUT | RELEASED | RECONCILIATION_REQUIRED`), ownership metadata (`payoutProvider`, `payoutProviderName`, `intendedProvider`), `relatedReservationReference`, terminal claim fields, linked `MerchantFeeGhs`/ledger references. **No `TransactionHistory` row, no fabricated USDC value, no personal user-wallet debit or credit** — the business's GHS payable balance is the only economic source. A merchant payout is **manual and merchant-initiated only** (owner-decided); automatic payouts are out of scope at launch.

**One immutable internal business identity.** The `businessProfileId` (the KYB business profile) is THE single canonical internal identity of the merchant counterparty. The payable account, the receivable account, every `MerchantPayout` row and every `MerchantPayoutDestination` key on it — never on the owner's `userId`, never on a wallet address, never on a provider-supplied identifier. Provider-side references (Moolre/MTN transaction or provider ids) CORRELATE to this identity and to the payout reference but can never redefine, replace or merge it: a provider response naming a different business or a different payout parks (`RECONCILIATION_REQUIRED`), it never rewrites ownership (the same §L-style substitution closure as the liquidity authority). A business whose profile is re-KYB'd or whose legal form changes keeps one identity; a NEW `businessProfileId` is a NEW counterparty with zero opening balance — no balance ever migrates between identities.

**The payout reference is immutable and platform-generated.** `MerchantPayout.reference` is created by the platform at initiation, is unique, never reused, and never changes across the lifecycle; provider references, dispatch evidence, the liquidity reservation reference, the §4.2/§4.4/§4.5 ledger idempotency keys and the terminal claim all CORRELATE to this one reference. `MerchantPayout` is therefore THE durable correlation record of the economic payout: one row per economic payout, linking every side effect, and nothing about that linkage may be edited after the fact.

### 6.2 Verified payout destinations

NEW `MerchantPayoutDestination` per business: `msisdn`, `beneficiaryName`, `verificationStatus`, `verifiedAt`, `verifiedBy/method`, `active`. Verification supports a **registered-business MoMo (KYB evidence)** or a **legitimate sole-proprietor's verified MoMo (KYC/KYB evidence of beneficial ownership)** — the contract does not assume every business owns a separate registered business number (owner-decided). Payouts dispatch only to an active, verified destination. Destination changes require re-verification, an audit record and four-eyes approval; payout initiation requires an authorized business-portal role and is fully audited (who, when, destination, amount).

### 6.3 r15 guarantees retained verbatim

Dispatch evidence before ownership persistence; fail-closed provider identity (no invented rail, no `|| 'MTN_MOMO'` fallback); one terminal winner; durable exceptions with the parking matrix (`PROVIDER_STATUS_UNRESOLVED`, `PROVIDER_REFERENCE_NOT_FOUND`, `PAYOUT_OWNERSHIP_CONFLICT`, `DISPATCHED_OWNERSHIP_NOT_DURABLE`); never refund/debit after provider-confirmed acceptance. The ownership module's semantics apply against `MerchantPayout` rows (same resolve/fail-closed logic; whether the module is parameterized or mirrored is an implementation choice, the semantics are identical and tested identically). Reconciliation reuses `reserveForPayout`'s single-winner reservation machinery for the GHS liquidity claim.

### 6.4 The exact-2dp boundary through provider dispatch — adapter contract fix required

The traced defect `String(Number(amountGhs))` (`services/moolreDisbursementService.js:269`) passes authoritative money through binary floating-point. **P6b implementation prerequisite:** the disbursement boundary must accept and serialize a **canonical exact-2dp GHS string** derived from the pesewas authority (integer pesewas or `Decimal.toFixed(2)`), validated against a strict `/^\d+\.\d{2}$/` pattern before any provider I/O; `Number()`/float arithmetic anywhere on an authoritative money path is rejected in review; provider minimum/maximum limits are enforced from configuration before dispatch. **Moolre MoMo is the only proposed launch payout rail; no bank payout is available or claimed.**

---

### 6.5 Fee-inclusive liquidity coverage — OPEN FINANCIAL DESIGN BLOCKER

The ledger settles asset outflows of `X + F` (payout, §4.4) and `R + F` (refund disbursement, §4.6) against `fiat:momo:ghs`, and `F` is recognized ONLY from durable provider evidence — it is never invented. But a reservation of the principal alone (`X` / `R`) leaves the pool projection overcommitted by exactly `F` whenever the provider charges a fee: the `SystemFiatPool` would report GHS as available that is in fact committed to the provider fee. Two candidate designs, both inside the P5-D authority and both requiring the provider fee schedule as a **commercial input that is still open** (§9):

1. **Fee-inclusive reservation (preferred candidate):** the payout/reservation claims `X + F_cap`, where `F_cap` is a per-rail, configuration-verified maximum provider fee. Dispatch fails closed if a provider-quoted fee exceeds `F_cap`. At settlement, the evidenced `F` reconciles the claim: `F_cap − F` is released back to AVAILABLE, so the projection is exact at every terminal state (PAID_OUT claims `X + F`; RELEASED claims nothing).
2. **Separately reserved, evidenced fee debit:** the principal claims `X`, and a distinct fee reservation (bounded by `F_cap`) claims the fee allowance; the provider-fee event — not an estimate — decrements it; unused allowance is released at settlement.

Both keep reserved ≥ evidenced outflow and never invent `F`. **Until one design is selected and the provider fee schedule is commercially confirmed, this reservation model is NOT complete for `F > 0`** — this section is an open financial design blocker on P6 settlement code (it is folded into gate 6). With `F = 0` (fee-free dispatch) the §5.2 model is complete as written.

## 7. Financial invariants — the durability-suite pins

Each pin becomes a real-PostgreSQL test in the financial-durability suite before any GHS settlement code merges:

1. **Per-order conservation:** `payable + fee == gross` for every confirmation posting (exact pesewas).
2. **No cross-asset leakage:** no GHS-paid order path ever posts `ASSET_CONVERSION` or mixes assets in one posting; payable/receivable/fee/clearing accounts never hold USDC.
3. **Exactly-once economics:** duplicate collection callbacks, duplicate payout dispatches, duplicate refund submissions and re-reconciled reversals each converge to one economic mutation (receipt dedupKey, ledger idempotencyKey, terminal single-winner claims).
4. **Single-winner transitions:** concurrent reservations/payouts/refunds race under real PG and produce exactly one winner; losers roll back completely (no partial ledger rows).
5. **No negative payable:** payable never crosses to debit balance; a refund beyond payable forces the durable receivable path (which itself blocks payouts until a four-eyes release).
6. **No duplicate GHS asset recognition:** event↔receipt↔ledger-entry 1:1:1 per collection reference; the deposit, treasury and order receipt classes are disjoint and cross-claiming fails closed.
7. **Projection equality:** `MerchantPayableSummary` equals ledger-derived balances per business, and `SystemFiatPool` equals the receipt/reservation authority (nightly proofs; mismatch parks an exception).
8. **r15 ownership compliance on merchant payouts:** evidence-before-ownership write order, fail-closed identity, parking matrix, no debit after confirmed acceptance.
9. **Fee invariants:** fee ≤ gross; schedule version recorded per fee; schedule changes never recompute historical fees; full-refund commission reversal restores the merchant's payable exactly (no hidden merchant charge).
10. **Refund invariants:** refund ≤ payment; refund currency == payment currency; every refund path balanced (§4.6/§4.7), including the receivable path; refund disbursement failure restores payable + commission exactly.
11. **Provider-evidence reconciliation:** UNKNOWN_OUTCOME never posts and never fails silently; late and contradictory callbacks park with durable exceptions and never settle or reverse a terminal row.
12. **Float-free money:** authoritative GHS values are exact-2dp strings end to end; any `Number()`-mediated serialization on a money path fails the review gate.
13. **Fee-coverage (blocked on §6.5):** total reserved (principal + fee coverage) ≥ evidenced asset outflow for every dispatch; the pool projection never understates committed GHS; the unused fee allowance is released exactly once at settlement. Testable once the §6.5 design is selected.

## 8. Release gates — required before GHS settlement is enabled

The contract and rollout must not be treated as satisfied until ALL of these exist and pass:

1. **This document signed off by the owner** (including the §9 decisions marked open).
2. **Complete balanced accounting examples** for collection, payment, fee, payout, refund, reversal and provider cost — provided in §4, pinned by the §7 tests that assert each example's conservation under real PG concurrency and durability (duplicate callbacks/payouts/refunds, single-winner transitions, no-negative-payable, receivable handling, reconciliation).
3. **No-duplicate-asset-recognition and ledger-vs-projection equality proofs** (§7.6, §7.7) as executable reconciliation tests.
4. **Provider evidence and reconciliation tests** covering unknown outcomes and late/contradictory callbacks (§7.11).
5. **Verified payout destinations and permissions** — destination verification, change control and payout authorization implemented and tested (§6.2) before the first live payout; four-eyes on first payouts.
6. **Explicit commission and fee terms** — the versioned GHS fee schedule, its percentage/basis and merchant terms approved as commercial terms, not implementation constants (§9) — **including resolution of the fee-inclusive liquidity coverage design (§6.5, pin 13): reservations must cover the full evidenced asset outflow, principal plus platform-absorbed provider fee.**
7. **Approved collection/settlement quote contract for USDC-paying customers** (§2.3): purpose-specific server quote, expiry, explicit confirmation — reviewed separately; the public oracle is never the checkout quote.
8. **Honesty boundary preserved:** GHS listing prices and discovery development remain distinct from GHS checkout claims. Until collection AND merchant settlement are implemented and verified, **no UI, API capability flag, or product copy may promise GHS order payment** (`payableIn` never reports "GHS" before that point).

Rollout order (unchanged): sign-off → catalog/schema PR (additive, no behavior) → GHS collection rail (own review) → settlement + payout PR (full §7 battery) → four-eyes monitored first payouts. Discovery/Mix GHS listing work proceeds independently and never gates on P6.

## 9. Decision status (owner, 2026-10-10)

**Decided:**
1. **Business-scoped GHS merchant liability:** APPROVED — the KYB business is the payable counterparty, not the owner user wallet.
2. **Automatic conversion of merchant GHS proceeds to USDC:** PROHIBITED unless a separately reviewed contract is explicitly approved.
3. **Merchant payouts:** manual, merchant-initiated only at launch; automatic payouts out of scope.
4. **Payout fees:** the platform absorbs provider disbursement costs initially, recorded to `expense:provider:fees:ghs`.
5. **Payout beneficiary:** business-authorized, verified MoMo destination; legitimate sole-proprietor verification is supported — not every business is assumed to own a separate registered business number.
6. **Free/zero-priced items:** unsupported for GHS Mix v1 until a separate model is approved.
7. **Full refunds:** commission reverses and all entries balance; a merchant is never charged a hidden additional amount.
8. **USDC payment for a GHS-priced order:** separate approved purpose-specific server quote with expiry and explicit customer confirmation; the public trading oracle is never the checkout quote.

**Still open (owner-only):**
1. **Sign-off of this contract** (rev 2), including the §2 scope amendment and the §4 postings.
2. **Minimum payout threshold:** must be configured; the numeric value remains open until actual provider fees, limits and operating costs are evaluated.
3. **Fee-inclusive liquidity reservation design (§6.5):** the two candidate designs are specified; selecting one requires the provider fee schedule (a commercial confirmation). Until then the reservation model is explicitly incomplete for `F > 0` and P6 settlement code stays blocked (gate 6, pin 13).
4. **Marketplace commission terms:** separate, versioned GHS schedule confirmed — but the percentage, fee basis and merchant terms remain open commercial approvals until explicitly set; the escrow fee rate must never be inherited accidentally.
5. **Collection/settlement quote contract for USDC-paying customers:** needs its own approval (gate 7).

— End of revision 2. Documentation only; no application source code changed, nothing deployed, nothing merged.
