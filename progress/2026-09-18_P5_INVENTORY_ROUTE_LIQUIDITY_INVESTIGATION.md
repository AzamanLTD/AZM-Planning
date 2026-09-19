# §P.5 Preparation — Inventory, Route-Aware Quotes, and GHS Liquidity

**Date:** 2026-09-18 UTC  
**Status:** P5-A / P5-B / P5-C / P5-D VERIFIED; NEXT IMPLEMENTATION BOUNDARY = §P.5-E MODEL B SETTLEMENT  
**Backend main:** `f3f72a652d2d361729214a48239a3cf08cf217ea`  
**Precondition:** §P.4 authoritative liability ledger merged and verified.

## 1. Purpose

§P.4 establishes the authoritative USDC customer-liability ledger and custody/evidence boundary. The next architecture slice is the asset/economic side:

1. Model B inventory and realized economics.
2. Route-aware quote authority.
3. GHS liquidity states and reconciliation.
4. Kotani Model A when its provider contract/evidence is ready.
5. Treasury location/movement controls and exchange abstraction later.

This document exists to prevent the next agent from treating these as unrelated CRUD features. They are one accounting chain:

`QUOTE → ROUTE DECISION → EXECUTION → ACTUAL COST/PROCEEDS → SETTLEMENT → REALIZED ECONOMICS`

## 2. Existing implementation facts

### Corporate purchase / inventory precursor

`CorporatePurchaseLog` exists with:

- `usdcAmount`
- `fiatSentTotal`
- `discountRate`
- `actualMarketRate`
- `purchaseMethod`
- `gatewayProvider`
- `gatewayReference`
- `adminId`
- `createdAt`

It is an audit log, not an authoritative inventory position. It has no lot identity, remaining quantity, cost basis ledger transaction, location, acquisition evidence state, consumption history, or realized P&L linkage.

Therefore do **not** reinterpret this table as inventory authority without an explicit migration contract.

### Fiat quote authority

`src/services/transactionQuoteService.js` now provides:

- persisted transaction quotes;
- user/purpose binding;
- consume-once semantics;
- TTL;
- canonical `liveRetailRate`;
- §P.1/271C fresh external-rate gating for deposit quotes;
- true `lastExternalSync` provenance.

But the quote model is still fundamentally a fiat-deposit quote. Route is not yet a first-class persisted decision. There is no authoritative:

`quote → chosen route/provider → execution attempt → actual provider result → realized spread/cost`

chain.

### Provider/rate surfaces

`services/oracleService.js` can obtain a Kotani USDC/GHS observation.

`services/gatewayService.js` supports Kotani off-ramp rate/payout/status behavior, but Kotani Model A on-ramp is not yet implemented.

Do not claim Model A merely because a Kotani rate API exists.

### GHS liquidity

`SystemFiatPool` is still a singleton operational balance used by the current fiat payout reservation path. It is not yet an evidence-backed GHS custody/liquidity state machine.

Current §P.4 fiat off-ramp deliberately uses:

`clearing:fiat:offramp:usdc`

rather than fake `custody:provider:usdc`.

The later GHS wave must introduce actual states/evidence such as:

`RECEIVED → AVAILABLE → RESERVED → IN_TRANSIT → PAID_OUT`

with reversal/reconciliation states, tied to real Moolre/MTN evidence.

## 3. Critical prerequisite discovered before Model B

The P4 `LedgerAccount` catalog already contains an `asset` attribute, including future accounts such as:

- `fiat:momo:ghs` — GHS asset
- `inventory:usdc:lots` — USDC inventory asset
- `user:{id}:liability` — USDC customer liability

However, the authoritative `ledgerService.post()` primitive currently balances the journal numerically across all lines but does not enforce that lines belong to the same asset identity.

That is safe only while all authoritative postings are USDC-denominated.

Model B is explicitly multi-asset:

- customer pays GHS;
- GHS asset increases;
- USDC inventory is consumed/acquired at a GHS cost basis;
- customer USDC liability changes;
- spread/cost realization occurs.

Do **not** implement this by putting raw GHS and USDC quantities into one numerically balanced journal. That would create a false accounting equality.

Before Model B, define one explicit cross-asset accounting contract:

### Preferred boundary

Use separate asset-balanced ledger books / postings with an explicit clearing/exchange identity connecting them.

At minimum:

- every authoritative posting line has a resolved asset identity;
- a posting group may not silently balance GHS against USDC;
- cross-asset exchange/conversion must carry an explicit conversion/exchange reference and two balanced asset-specific legs;
- quantities remain exact decimals;
- the conversion relationship, rate, quote identity and actual execution evidence are persisted;
- no realized spread is created merely from a quote;
- expected economics and realized economics remain distinct.

This is a prerequisite for inventory economics, not an optional cleanup.

## 4. Target Model B shape

For a GHS→USDC customer purchase:

### Acquisition / liquidity side

Real GHS receipt:

- authoritative GHS asset/liquidity record;
- provider evidence;
- receipt reference;
- exact GHS amount;
- settlement state.

Inventory:

- immutable inventory lot;
- USDC quantity;
- GHS acquisition cost;
- source/evidence;
- remaining quantity;
- location;
- acquisition ledger identity.

### Customer settlement

Customer USDC liability is credited only from an authoritative inventory/cost-backed route.

The system must know:

- quoted GHS amount;
- quoted USDC amount;
- selected route;
- inventory lot(s) consumed;
- actual acquisition cost;
- actual settled amount;
- realized customer spread;
- provider cost/fee;
- remaining lot quantities.

The system must never manufacture inventory simply because a fiat deposit webhook completed.

## 5. Inventory design constraints

The first inventory implementation should favor immutable evidence and deterministic consumption:

- lot quantity is exact;
- remaining quantity cannot exceed original quantity;
- consumed quantity cannot become negative;
- every consume/release operation is atomic;
- duplicate consumption has a durable economic identity;
- inventory movements link to ledger transaction IDs;
- historical lots are never silently rewritten;
- lot cost basis is never inferred from the current market quote;
- realized P&L is produced only from actual settled execution;
- reconciliation can prove:
  `sum(open lot remaining) + sum(consumed movements) = sum(acquired quantities)`
  subject to explicit adjustment/reversal states.

The cost-flow policy (for example FIFO versus another explicitly approved policy) must be documented before it becomes economic authority. Do not invent a policy merely to complete a schema.

## 6. Route-aware quote contract

Extend the quote authority only after preserving current 271C freshness behavior.

A future quote should identify:

- purpose;
- input asset/currency;
- output asset/currency;
- customer amount;
- provider/route candidate set;
- selected route;
- quote source;
- external observation timestamp;
- quote expiry;
- expected provider fee;
- expected spread;
- route-policy version;
- deterministic quote identity.

At settlement, the provider/execution result must be matched to the quote and the actual economics calculated from evidence.

A mock route must not be allowed to masquerade as a live external route.

## 7. GHS liquidity state machine

Do not continue extending `SystemFiatPool.balance` as if it proves cash custody.

Create an evidence-backed liquidity model that can reconcile:

- Moolre receipt;
- banking/fiat custody evidence where available;
- provider payout reservation;
- in-flight payout;
- successful payout;
- reversal;
- unmatched/exception state.

The operational singleton can remain a compatibility projection until the new state machine becomes authoritative.

## 8. Exact implementation sequence

### P5-A — Multi-asset ledger contract

**VERIFIED — PR #280 squash `ab387e77`; exact-head CI #1046.** Small prerequisite slice.

Implement and test the accounting identity boundary without changing production money flows:

- resolve account asset metadata;
- reject a posting that numerically balances but mixes incompatible assets without an explicit exchange/conversion context;
- preserve every existing USDC P4 path;
- add real-Postgres proof cases;
- do not mutate production balances;
- stay within the <=500 changed-line PR rule.

### P5-B — Inventory lot foundation

**VERIFIED — PR #281 squash `39a4977`; exact-head CI #1048; post-merge main CI #1049.** Add the authoritative lot/movement schema and service, but do not wire customer deposits until evidence/route settlement is complete.

### P5-C — Route-aware quote

**VERIFIED — PR #284 squash `19671d7`; exact-head CI #1055; post-merge main CI #1056.** Add route/provider identity and route policy to the existing TransactionQuote lifecycle while preserving freshness/idempotency.

### P5-D — GHS liquidity state machine

**VERIFIED — PR #285 squash `baaa4e2`; exact head `22ebebd`; CI #1060; 55 dedicated real-PostgreSQL proofs; full battery 227/227 suites, 1719/1719 tests.** The evidence-backed authority now replaces the conceptual singleton as financial authority while retaining `SystemFiatPool` only as a compatibility projection. Recorded-row regime semantics and exact reserved-GHS payout dispatch are enforced.

### P5-E — Model B settlement

**NEXT.** Only after A-D prove the accounting identities, link GHS receipt → inventory lots → customer liability → realized spread. This slice must use actual settled GHS evidence, authoritative inventory lot consumption, explicit asset conversion, and realized economics; no quote-only P&L, synthetic GHS or synthetic inventory.

### P5-F — Model A

Implement Kotani on-ramp only from its actual provider contract and evidence; keep live provider execution gated OFF until separately authorized/tested.

## 9. Non-negotiables

- No production financial-data correction.
- No synthetic inventory.
- No synthetic GHS cash.
- No quote-only P&L.
- No cross-asset journal balancing by numeric coincidence.
- No live crypto signing/broadcast changes.
- Preserve P4 fail-closed behavior and all exact CI/recovery gates.
- Do not reopen completed P1-P4 authority work without a demonstrated regression.

## 10. Evidence for completion

Every P5 slice needs:

- real-PostgreSQL tests for economic invariants;
- exact-head CI;
- route verification where APIs change;
- Prisma validation;
- production dependency audit;
- backup/restore drill for schema/state changes;
- diff audit against this document;
- verification on backend `main`;
- update to `CURRENT_STATE.md`, `ACTIVE_LOOP.md`, and `EXECUTION_LEDGER.json`.



## 11. Verified P5-C / P5-D completion evidence

### P5-C

PR #284 is the accepted route-aware quote re-land from the clean post-revert baseline. Final exact head `4b365f7`; squash merge `19671d7`; exact-head CI #1055 and post-merge main CI #1056 succeeded. Route/provider/rail identity is persisted and settlement-bound, quote replay/conflict behavior is deterministic, exact decimal rate provenance is preserved, and §271C external-rate freshness remains authoritative.

### P5-D

PR #285 final exact head `22ebebd`; squash merge `baaa4e2`; exact-head CI #1060 succeeded. The dedicated P5-D suite contains 55 real-PostgreSQL proofs and the clean full battery is 227/227 suites and 1719/1719 tests, with Prisma validation, route-check, dependency audit and recovery drill successful. The final worker contract is recorded-row based: reservation-backed rows dispatch the exact reserved GHS despite rate drift; legacy unreserved rows keep legacy pool semantics; authority rows never consult `SystemFiatPool` as an input and never create a second reservation. No production financial data changed. The available GitHub workflow lookup cannot independently expose a post-merge push-triggered CI result for merge SHA `baaa4e2`, so no such result is claimed.

**Next implementation boundary:** §P.5-E Model B settlement / cost-basis realization.
