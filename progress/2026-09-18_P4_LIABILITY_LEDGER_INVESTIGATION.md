# §P.4 — Authoritative Liability Ledger & Custody/Financial Authority Investigation

**Date:** 2026-09-18 UTC  
**Status:** READY FOR IMPLEMENTATION  
**Baseline:** `AzamanLTD/AZM-backend` main `be6038f0528b9e7c62a203e99708f16198a5d898`  
**Purpose:** Define the exact safe transition from legacy user-balance authority + shadow journal to one authoritative, transactional customer-liability ledger without duplicating the financial engine or mutating production financial data.

## 1. Verified baseline

§P.1 WalletAddress authority, §P.2 KMS-capable custody execution, and §P.3 custody accounting/evidence are merged on backend main.

§P.3 deliberately leaves:
- `User.availableBalance`, `escrowLockedBalance`, `disputeEscrowBalance`, and `vendorUnallocatedBalance` as live materialized financial projections;
- `TransactionHistory` as the operational customer-facing transaction/settlement record;
- `JournalEntry` as a shadow double-entry record;
- `SystemMasterCrypto`, `SystemHotWallet`, `SystemFiatPool`, and `SystemProfitFees` as legacy display/bookkeeping singletons rather than custody truth;
- restricted obligations as explicitly unmodelled.

No historical production balances have been corrected or backfilled by §P.3.

## 2. Newly verified P4 blockers

### 2.1 Journal atomicity is false today

`services/journalService.js` constructs its own module-level `PrismaClient` and `record()` runs a separate Prisma transaction. Several financial controllers mutate money inside their own caller-owned `prisma.$transaction`, then invoke journal recording later or asynchronously.

Examples:
- crypto deposit commits user balance + TransactionHistory + custody candidate, then calls `journal.recordDeposit(...).catch(...)` after commit;
- crypto withdrawal commits customer debit + execution reservation, then calls `journal.recordWithdrawal(...).catch(...)` after submission;
- trade escrow lock records the journal after the balance mutation and does not block settlement on journal success.

Therefore a journal failure cannot roll back an already committed financial mutation. This is incompatible with authoritative-ledger semantics.

### 2.2 Journal numeric validation is not exact

`journalService.record()` validates balance using JS `parseFloat` plus an epsilon and persists parsed floats. P4 must make all authoritative ledger arithmetic exact via `Prisma.Decimal` / exact decimal strings. No binary-float ledger quantity may remain authoritative.

### 2.3 Existing journal account semantics are wrong for customer liabilities

The current helpers treat `user:{id}:available` as an asset account whose debit increases a user's balance. The target architecture is liability-aware:
- customer USDC liability increases on a credit to the user liability account;
- customer liability decreases on a debit;
- custody USDC assets increase with debits and decrease with credits.

P4 must not preserve the old asset interpretation under a cosmetic rename.

### 2.4 Journal coverage is incomplete and non-authoritative

Major financial mutations still depend on User bucket columns and/or singleton balances without a mandatory ledger posting in the same database transaction. Known affected families include crypto/fiat deposits, fiat/crypto withdrawals, P2P transfer, trade escrow, SmartEscrow, booking escrow, vault, Susu, invoice settlement, and related refund/reversal flows.

P4 must build a repo-wide mutation matrix before changing callers and then migrate the relevant financial paths through the canonical ledger primitive.

## 3. Target authority model

### 3.1 One accounting truth

The backend must have one authoritative double-entry ledger. Extend the existing JournalEntry foundation; do NOT create a second independent ledger.

A clean implementation may add a ledger-account catalog and a ledger-transaction/posting-group parent if necessary to enforce:
- explicit account classification/normal side;
- one durable economic transaction identity;
- idempotent replay;
- deterministic line ordering;
- immutable posted entries;
- exact Decimal quantities.

Those are ledger metadata/support structures, not a parallel financial engine.

### 3.2 Account semantics

At minimum support the canonical chart:

- `custody:deposit:usdc` — asset
- `custody:hot:usdc` — asset
- `custody:cold:usdc` — asset (future activation)
- `custody:exchange:usdc` — asset (future activation)
- `custody:provider:usdc` — asset
- `fiat:momo:ghs` — asset (future GHS-liquidity wave)
- `inventory:usdc:lots` — asset (P5)
- `user:{id}:liability` — customer liability
- `escrow:{id}:locked` — restricted customer/escrow liability classification
- `restricted:reserves` — restricted obligation liability (P4 must establish semantics)
- `clearing:conversion` — conversion clearing
- `revenue:fees` — revenue
- `revenue:spread` — revenue
- `expense:gas` — expense
- `expense:provider` — expense
- `pnl:inventory` — P&L (P5)
- `pnl:arbitrage` — P&L (later, exchange wave)
- `equity:treasury` — equity

Do not invent realized economics before actual provider/inventory evidence exists.

### 3.3 Customer balance columns are projections

Keep the existing User balance columns during migration so existing API consumers and UI stay compatible. They become materialized projections of authoritative ledger state.

Every mutation must update:
1. authoritative ledger postings;
2. User bucket projection(s), when the operation changes a bucket;
3. TransactionHistory / domain state where applicable;
4. audit/event state where applicable;

inside the same DB transaction.

No ledger-only or projection-only financial mutation is acceptable on migrated paths.

### 3.4 TransactionHistory is not the ledger

`TransactionHistory` remains the customer-facing business event + provider settlement record. It must not be used as the accounting balance engine.

Provider lifecycle states remain authoritative for provider settlement, while the ledger records the resulting economic state transitions.

## 4. Expand → migrate → contract sequence

### Phase A — Expand, no economics change

Add the minimum additive schema needed to make ledger authority enforceable:
- explicit ledger account metadata/classification;
- durable ledger economic transaction/posting group with unique idempotency identity;
- immutable posting lines / line ordering;
- nullable migration links on existing JournalEntry rows where required;
- indexes/uniqueness needed for exact replay;
- no historical deletion.

Write an idempotent installer/migration safe for populated production databases. Do not modify financial balances.

### Phase B — Canonical transactional posting primitive

Create one canonical service boundary, e.g. `ledgerService.post(tx, {...})`, that:
- requires caller-owned Prisma transaction client for financial mutations;
- validates authorization context supplied by caller but never owns business authorization;
- validates all amounts as exact `Prisma.Decimal` values;
- requires balanced total debits == credits using Decimal arithmetic;
- refuses zero-value lines/negative line quantities where policy forbids them;
- records one economic transaction identity;
- is idempotent by durable unique key;
- returns the existing committed transaction/result on exact replay;
- fails the entire caller transaction if posting fails;
- never starts an independent database transaction underneath a caller-owned one.

A compatibility `journalService.record()` wrapper may remain temporarily for non-authoritative historical/test callers, but migrated financial paths must use the transaction-scoped authoritative primitive.

### Phase C — Ledger authority for external customer-flow edges

First migrate the simplest high-value edges because they define the authority boundary:
1. crypto deposit: D eligible custody asset / C user liability;
2. crypto withdrawal reservation/settlement: customer liability reduction and custody movement linkage without synthetic singleton authority;
3. fiat-settled USDC deposit: C user liability against an explicit conversion clearing account until P5 inventory economics exists;
4. fiat withdrawal reservation/settlement: reduce customer liability at the correct economic point; keep provider settlement deferred economics semantics unchanged;
5. internal user transfer: liability reclassification between sender and receiver with no fake external revenue.

Then migrate escrow/vault/Susu/invoice/booking/P2P family operations using the same primitive.

### Phase D — Reconciliation

Create deterministic reconciliation read models/checks that compare:
- authoritative ledger customer liability by user;
- Sum of materialized User bucket balances;
- TransactionHistory external-flow totals;
- CustodyMovement / CustodyExecution economic totals;
- restricted obligations;
- conversion clearing balance.

Reconciliation must report discrepancies; it must not silently rewrite financial history.

### Phase E — Contract and demotion

Once migrated paths are proven:
- all targeted financial writers must use ledger primitive;
- direct financial writes bypassing ledger must be prohibited by tests/static checks where practical;
- singleton balances become display-only projections/read models;
- old journal helper semantics are removed or clearly legacy-scoped;
- production data is not automatically corrected. Any repair requires explicit authorization and a separately audited repair plan.

## 5. Restricted obligations — P4 requirement

P4 must establish an authoritative persisted representation for restricted obligations so Proof of Reserves can eventually evaluate:

`eligible real USDC assets >= customer USDC liabilities + restricted obligations`

Do not store restricted obligations as a hard-coded zero.

At minimum define:
- obligation identity/reference;
- source domain (escrow, dispute, pending withdrawal, regulatory/operational reserve, etc.);
- asset and network identity;
- amount;
- status/lifecycle;
- whether/why the obligation is included in reserve denominator;
- linkage to the originating ledger/transaction/domain object;
- immutable or append-only history.

Where an existing domain already owns the obligation (e.g. an active escrow/dispute), link to that authoritative state instead of copying an independent balance that can drift.

P4 must preserve the explicit fail-closed behavior: until the denominator is complete, PoR cannot claim fully backed.

## 6. Custody linkage

P3 custody truth remains:
- CustodyAccount = authoritative custody location identity;
- CustodyMovement = custody movement truth;
- CustodyEvidence = external evidence;
- CustodyExecution = external signing/broadcast state.

P4 connects these custody facts to accounting entries without duplicating movement rows or JournalEntry economics.

For an on-chain customer deposit:
- CustodyMovement VERIFIED remains the custody truth;
- ledger customer liability is the customer obligation;
- custody asset account is the accounting asset representation;
- TransactionHistory remains the customer-facing event.

For withdrawals:
- no synthetic SystemHotWallet increment/decrement may remain the authoritative custody asset movement;
- link the ledger economic transaction to the CustodyExecution/CustodyMovement identity.

## 7. Critical safety rule

Do not perform a “historical backfill” that guesses opening balances, invents custody, invents inventory, or manufactures journal entries from incomplete history.

For populated production data, use one of:
- provably reconstructable historical posting from existing authoritative records; or
- an explicit opening/transition balance that is marked as an audited migration adjustment and remains outside normal operational economics.

Because production financial mutation is not authorized, default implementation should establish schema + transactional posting + reconciliation + migration tooling, with production execution of financial backfill OFF.

## 8. Real-PostgreSQL proof requirements

New P4 tests must prove, on real PostgreSQL:

1. balanced posting is exact Decimal arithmetic;
2. unbalanced posting rolls back the enclosing financial transaction;
3. journal failure rolls back the user balance/domain mutation;
4. two concurrent identical requests produce one economic ledger transaction;
5. two concurrent spend requests cannot produce negative projection or negative liability;
6. ledger replay returns the original committed result;
7. internal transfer posts balanced sender/receiver liability entries exactly once;
8. crypto deposit links liability + custody asset exactly once;
9. fiat-settled USDC credit links liability to conversion clearing, with no fabricated custody asset;
10. withdrawal reservation/settlement/reversal keeps ledger, User projection, TransactionHistory, CustodyExecution, and CustodyMovement coherent;
11. escrow/vault/refund reclassification does not change total customer liability incorrectly;
12. restricted obligation creation/release changes the reserve denominator deterministically;
13. legacy singleton rows are excluded from authoritative ledger/reserve calculations;
14. migration installer is idempotent and backup/restore safe;
15. no migrated financial route can commit money while its authoritative ledger posting is absent.

The full existing battery must remain green plus the new real-PostgreSQL P4 suite. Route-check, Prisma validation, dependency audit, and backup/restore drill remain required before merge.

## 9. Explicit non-goals for P4

Do NOT activate or implement:
- production crypto signing/broadcast;
- real testnet execution;
- Kotani Model A;
- Model B realized inventory economics;
- exchange trading/arbitrage;
- GHS liquidity state machine;
- automatic production balance repair.

Those remain later waves or separately authorized production activities.

## 10. Required implementation outcome

At the end of §P.4, the system must be able to answer from one authoritative accounting source:

- how much USDC each customer is owed;
- how much is available vs escrow/restricted by domain state;
- which ledger transaction created/removed that obligation;
- which custody asset movement/evidence backs the asset side;
- which obligations are restricted;
- whether User materialized balance projections reconcile exactly.

The system must fail closed whenever those questions cannot be answered deterministically.

