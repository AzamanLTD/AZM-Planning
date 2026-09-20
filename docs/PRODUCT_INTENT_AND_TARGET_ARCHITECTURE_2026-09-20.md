# AZAMAN Product Intent & Target Architecture Contract

**Status:** ACTIVE — canonical product-direction and architecture memory  
**Date:** 2026-09-20 UTC  
**Repository:** `AzamanLTD/AZM-Planning`  
**Purpose:** Preserve the product intent, financial semantics, identity model, scheduled-payment vision, and cross-repository implementation direction established during the 2026-09-20 deep architecture audit.

> This document is durable engineering memory. Future agents must read it alongside `ROADMAP.md`, `ARCHITECTURE.md`, `CURRENT_STATE.md`, and the relevant dated investigations before proposing or implementing work.

---

## 1. Product north star

AZAMAN is intended to become a working Ghana-focused financial/productivity platform that can safely be used with real customer money.

The engineering target is not simply a collection of features that pass happy-path tests. The target is a platform where:

- money has one authoritative accounting meaning;
- financial state is deterministic under retries, concurrent requests, provider ambiguity, crashes and reconnects;
- customer identity is explicit before value moves;
- internal transfers and external withdrawals are distinct concepts;
- business and consumer experiences share the same backend authority;
- recurring/scheduled payments are a generalized product primitive rather than a collection of special cases;
- QR/tap payment requests never authorize money movement by themselves;
- business payroll/EWA uses real monetary balances, not loyalty points;
- provider integrations are represented as rails/execution boundaries, not leaked as product semantics;
- audit/reconciliation can explain where every economically meaningful balance came from and where it went.

**Never optimize for speed of implementation at the cost of financial ambiguity.**

---

## 2. Repository truth and terminology

AZAMAN is one distributed product.

| Repository | Product role |
|---|---|
| `AzamanLTD/AZM-backend` | Server authority: identity, authorization, money, ledger, state machines, APIs, events, reconciliation |
| `AzamanLTD/AZM-frontend` | Consumer/customer Flutter application |
| `AzamanLTD/AZM-businessPortal` | Business operating/control web portal |
| `AzamanLTD/AZM-adminPortal` | Privileged administration/governance web portal |
| `AzamanLTD/AZM-Planning` | Engineering memory, contracts, roadmap, research and evidence |

**Important terminology rule:** `AZM-businessPortal` is NOT the consumer frontend. The consumer frontend is `AZM-frontend`.

Current CI route-check coverage that targets the Business Portal must not be mistaken for complete Flutter API compatibility coverage. A future consumer API compatibility gate is still required.

---

## 3. Canonical financial rail model

### 3.1 Current external fiat provider

**Moolre is the only current external fiat provider contract for AZAMAN.**

AZAMAN does not currently hold a direct MTN API contract.

MTN, Telecel/Vodafone and other local network/operator labels may appear as destination-network information that Moolre supports, but they are not independent AZAMAN provider abstractions.

Therefore:

- production fiat on-ramp/collection = Moolre;
- production fiat off-ramp/disbursement = Moolre;
- local network selection belongs to the Moolre destination payload/domain model;
- the product should not expose “MTN” as though it were a separate AZAMAN provider;
- existing `mtn*` service/adapter naming is architectural leakage to be audited and progressively removed or reduced to compatibility aliases/internal network terminology.

### 3.2 Withdrawal classes

There are two externally directed withdrawal families:

1. **Crypto withdrawal:** USDC to an external crypto destination through the canonical crypto custody/execution boundary.
2. **Local fiat withdrawal:** USDC converted to GHS and paid through Moolre to a supported MoMo/bank destination.

Do not create separate AZAMAN providers for each mobile network.

### 3.3 Internal movement is not withdrawal

Internal payments are:

- **AZM-ID → AZM-ID** for user-to-user payments.
- **AZM-ID → BIZ-ID** for user-to-business payments.

Internal payments are not off-ramp operations and must not be modeled as Moolre withdrawals.

---

## 4. Customer money vs AZM loyalty points

This distinction is non-negotiable.

### Monetary balance

`User.availableBalance` represents customer spendable monetary value (currently USDC in the main financial model) and is a materialized projection of the authoritative financial liability ledger.

### AZM loyalty balance

`User.azmBalance` is an independent AZM loyalty-point balance.

It is **not** the customer's spendable USDC balance.

Therefore:

- payroll money must not be credited to `azmBalance`;
- EWA advances must not be credited to `azmBalance`;
- customer deposits must not be represented as loyalty points;
- financial accounting must not use loyalty balances as reserve/liability substitutes;
- any code that treats `azmBalance` as salary, EWA or other customer cash is a P0 semantic defect.

Known audit findings to close:

- `PayrollService.disbursePayroll()` currently credits `azmBalance` with amounts documented as USDC.
- `EwaService.requestWithdrawal()` currently credits `azmBalance` with employee advance amounts.
- These flows must be migrated to the canonical monetary liability path, with corresponding business funding/source-of-funds accounting.

---

## 5. Identity primitives

### 5.1 AZM-ID

AZM-ID is a human-facing stable user identity.

Backend numeric `User.id` is an internal database identifier and must not remain the primary customer-facing payment identity.

Customer-facing payment/search/association flows should converge on AZM-ID.

### 5.2 BIZ-ID

BIZ-ID is a human-facing stable business identity.

`BusinessProfile.bizId` is the canonical business identifier and should be the customer-facing identity for user→business payment and business discovery.

### 5.3 Identity resolution service

Create a canonical identity-resolution layer that can resolve:

- AZM-ID → user;
- BIZ-ID → business;
- verified phone/contact identity → user, subject to privacy/opt-out/block rules.

Resolution must return a safe **recipient preview** before payment authorization, including as appropriate:

- display name;
- AZM-ID or BIZ-ID;
- verified state;
- avatar/business logo;
- business/user distinction;
- other non-sensitive disambiguating information.

**Security rule:** globally unique identifiers are identifiers, not authorization. Authorization must still enforce tenant, actor and destination ownership rules.

### 5.4 Payment authorization sequence

The canonical sequence is:

**Resolve identity → preview recipient → show amount/fee/context → explicit payer confirmation → execute authoritative transaction → generate receipt → emit post-commit convergence event.**

Do not allow stale friendship records, raw numeric IDs, QR contents, or local cached identity to silently substitute for current canonical resolution.

### 5.5 Business ownership semantics

For `AZM-ID → BIZ-ID` payment, the business should eventually behave as its own financial participant/account rather than blindly crediting the business owner's personal user balance.

This must be audited and designed against the liability ledger before broad rollout.

---

## 6. Phone contact discovery and social graph

The product should support a natural, WhatsApp-like experience for people who already know each other's phone numbers without forcing a formal friend-request workflow for ordinary chat access.

Current architecture already has the basis:

- device-side phone hashing;
- `User.phoneHash`;
- `phoneVerified`;
- contact-discovery opt-out;
- `/api/contacts/sync`;
- matched AZM-ID data.

### Target behavior

Phone discovery should identify known contacts without exposing raw phone numbers.

Chat access may be allowed based on the contact relationship while preserving:

- block controls;
- privacy controls;
- contact-discovery opt-out;
- no raw-number disclosure.

**Important separation:** chat eligibility is not financial authorization.

A contact relationship must never authorize a payment by itself.

---

## 7. Generalized scheduled payments / payout plans

The current Smart Route implementation is too narrow for the intended product.

The long-term product is a **generalized scheduled payment / payout plan engine**.

It should support at least:

- child weekly allowance;
- recurring transfer to an AZM-ID;
- recurring local-fiat withdrawal through Moolre;
- recurring business/worker payout;
- salary-driven business payouts;
- recurring savings or vault actions only where the financial semantics remain appropriate;
- grouped business payouts;
- role/tag/rank-based payout rules;
- individual amount overrides;
- bonuses;
- notes.

### 7.1 Business payout lists

A business owner should be able to:

1. select a set of workers/recipients;
2. give the list a human-readable name;
3. assign individual amounts;
4. define role/tag/rank rules such as “all MANAGERs receive X”;
5. add recipient-specific bonuses or overrides;
6. attach payout notes;
7. schedule execution;
8. approve/review where policy requires;
9. see execution batches and per-recipient outcomes.

### 7.2 Suggested domain model

Do not force the final schema without a complete authority audit, but the conceptual model should be close to:

- `PaymentPlan` — owner, name, schedule, status, policy;
- `PaymentTarget` — internal user, internal business, Moolre destination, external crypto destination;
- `PaymentPlanRecipient` — recipient identity + configured rule;
- `PayoutRule` — fixed amount, role/tag/rank rule, effective dates;
- `PayoutOverride` — individual amount/bonus/temporary exception;
- `PayoutBatch` — one scheduled execution occurrence;
- `PayoutItem` — one recipient movement inside a batch;
- `PayoutApproval` — optional approval/four-eyes state;
- immutable execution/audit snapshot fields.

The schema names are not yet frozen. The **semantics** are.

### 7.3 Execution rules

At execution time the engine must snapshot the relevant recipient identity, rules and amounts so later edits cannot silently rewrite historical execution meaning.

The scheduler and manual “run now” path must converge on the same atomic execution boundary.

Every occurrence needs:

- single-winner claim;
- idempotency;
- deterministic retry behavior;
- concurrency protection;
- partial-failure semantics;
- reconciliation;
- canonical receipt/audit data.

### 7.4 Smart Route migration direction

Smart Routes should become either:

- a compatibility façade over the generalized scheduler, or
- a formally deprecated legacy domain once all relevant clients migrate.

Do not build an ever-growing list of Smart Route special cases.

Known current risks:

- scheduled/manual `runOnce` is read-then-act and can overlap;
- current Smart Route MoMo execution creates financial records before provider outcome and uses an MTN-named service path;
- Smart Route withdrawal/history reconciliation can lose the canonical bridge;
- `destFriendUserId` is not an AZM-ID destination;
- current provider/network labels need migration to the Moolre rail model.

---

## 8. Employee enrollment, worker experience, payroll and EWA

### 8.1 Employee enrollment

Business Portal should allow business owners to add employees by AZM-ID.

The canonical sequence is:

**Enter/search AZM-ID → resolve user → show employee identity preview → business confirms association → backend creates BusinessEmployee relation.**

A current integration defect exists:

- Business Portal employee creation sends `{ azmId: ... }`;
- backend `EmployeeService.addEmployee()` currently expects `userId` and does not resolve the AZM-ID.

This must be fixed at the authoritative backend boundary rather than worked around in the portal.

### 8.2 Worker portal

Once associated with a business, the employee should see the worker/employee experience in Flutter.

Existing Worker Payroll and Worker EWA surfaces are intended to consume authoritative backend data.

### 8.3 Payroll

Payroll records are currently documented as USDC amounts.

Payroll must therefore:

- debit/reserve/recognize the business's authoritative source of funds;
- credit the employee's monetary liability/balance through the canonical ledger;
- record gross/net/EWA/tax/bonus/tips/deductions consistently;
- reset accrued wage state safely;
- remain concurrency-safe and idempotent;
- expose authoritative payroll history to the worker and business.

Do **not** route monetary payroll through AZM loyalty accounting.

### 8.4 Earned Wage Access (EWA)

Business EWA is an advance of accrued wages. Conceptually:

**Employer employment/wage accrual → employee requests eligible advance → Azaman fronts/settles monetary value → employee receives monetary balance → employer payroll later settles the obligation.**

Required semantics:

- employee eligibility;
- maximum advance policy;
- minimum withdrawal policy;
- configurable fee;
- concurrency-safe accrued-wage reservation;
- business-side obligation/receivable;
- employee monetary credit;
- payroll settlement interaction;
- idempotent approval/execution;
- authoritative ledger entries;
- auditable fee/economics.

Current EWA has concurrency protection, but its financial credit target is wrong (`azmBalance`) and the fee is hardcoded at 1%.

### 8.5 Admin-configurable EWA policy

The EWA fee and relevant limits should be administered through the Admin Portal's authoritative policy/configuration layer rather than hardcoded in backend/frontend.

The exact admin policy schema can be designed during implementation.

---

## 9. QR payments and future tap-to-pay

### 9.1 QR payment request

A device must be able to create a payment request where:

- payee is known from the current authenticated session;
- amount may be specified or left open;
- request gets a short-lived signed token/request ID;
- QR encodes the request identifier, not authority to move money.

Scanner flow:

**Scan → backend resolves live payment request → resolve current recipient identity → display recipient + amount + fee → payer explicitly confirms → authoritative payment transaction → receipt.**

### 9.2 Security rule

**A QR code must never itself authorize money movement.**

No bearer QR should contain enough authority to silently debit a payer.

The backend remains authoritative for:

- request validity;
- expiry;
- payee;
- amount;
- currency;
- fee;
- payer authorization;
- idempotency;
- final settlement state.

### 9.3 Receipts

A canonical receipt should be able to include:

- payer;
- payee;
- AZM-ID/BIZ-ID;
- amount and currency;
- fee;
- exchange rate where relevant;
- rail;
- external destination where appropriate;
- transaction/reference ID;
- timestamp;
- status.

Tap-to-pay/NFC-like experiences should reuse the same payment-intent/authorization primitives rather than creating a second financial system.

---

## 10. Internal transfer architecture

Internal transfers should evolve from friendship/numeric-ID oriented behavior toward first-class identity-based payment destinations.

Target destination kinds include:

- `INTERNAL_USER`;
- `INTERNAL_BUSINESS`;
- `MOOLRE_MOMO`;
- `CRYPTO_EXTERNAL`.

The exact enum names may change, but the separation of **destination class** from **external execution rail** must remain.

Recipient identity and amount must be confirmed before execution.

Receipts and ledger/audit records should preserve an immutable execution-time identity snapshot.

---

## 11. Moolre architecture rule

Moolre is the local-fiat execution rail.

The application/domain layer should speak in terms such as:

- local-fiat deposit;
- local-fiat withdrawal;
- Moolre destination;
- supported mobile-network destination;
- bank destination.

Do not proliferate product-level branches such as:

- “MTN withdrawal”;
- “Vodafone withdrawal”;
- “Telecel withdrawal”

as independent providers unless an actual independent provider contract is introduced later.

Provider-specific implementation belongs behind the Moolre rail adapter.

---

## 12. Financial authority and ledger semantics

The authoritative accounting direction is:

**Customer balances = liabilities.**  
**Custody/inventory/GHS/external holdings = assets.**

The caller's transaction boundary must own the authoritative ledger writes.

Materialized User/domain balances may remain for performance, but they are projections and must be reconciled to the ledger.

### Required mutation pattern

`request context → identity/authorization → tenant/object scope → atomic claim/idempotency → transaction/conditional transition → ledger + projection + domain state → commit → event → client reconciliation`

No financial mutation should rely on:

- JS floating-point calculations for authoritative amounts;
- post-commit shadow journals;
- client-calculated money;
- socket payloads as truth;
- synthetic reserve balances as custody evidence;
- generic state patches that bypass financial lifecycle rules.

---

## 13. Provider execution and Moolre/crypto separation

There are two fundamentally different execution boundaries:

### Fiat

**Moolre** handles local-fiat provider execution.

Provider acceptance, processing, success, failure and ambiguity must flow through the canonical provider-execution/reconciliation model.

### Crypto

Crypto withdrawals/sweeps use the canonical custody execution boundary, currently designed around Tatum/KMS capabilities where enabled.

A crypto withdrawal is not a Moolre operation.

The customer-facing system should not let provider names leak into business semantics beyond what is needed for destination/receipt clarity.

---

## 14. Business payroll/payout relation to the generalized scheduler

Business payroll and scheduled payouts are related but not identical.

Payroll is an employment/accounting domain.

The generalized scheduled payout engine is the execution/orchestration layer.

Therefore:

- payroll calculates authoritative compensation;
- payroll determines obligations and payable amounts;
- a payout plan may orchestrate recurring payroll-driven execution;
- worker preferences such as balance/MoMo/wallet/split may become payout destination policy;
- execution must still run through canonical financial primitives;
- salary groups/ranks/tags are rules, not a second ledger.

Do not duplicate monetary settlement logic between PayrollService, SmartRoute and future PayoutService.

---

## 15. Known high-risk implementation backlog

These are architecture-level work items, not permission to skip the required research pass.

### P0 financial correctness

1. Payroll monetary credit currently targets AZM loyalty balance instead of monetary liability.
2. EWA monetary credit currently targets AZM loyalty balance.
3. Smart Route scheduled/manual execution is vulnerable to overlapping execution.
4. Smart Route MoMo reconciliation lacks a canonical Withdrawal ↔ TransactionHistory execution identity.
5. Admin withdrawal rejection must not refund customer money after provider execution may have escaped Azaman control.
6. Vault release must have one terminal winner under concurrent break-early/matured release.
7. Savings partial/full withdrawal/deposit operations require atomic claims against stale goal state.
8. Order Book BUY placement and matching require runtime and concurrent-consumption hardening.
9. Shared Vault is currently a stale/legacy route surface and must be reconciled against current authoritative financial architecture rather than resurrected through invented schema.

### P1 architecture/convergence

10. AZM-ID/BIZ-ID need to become first-class unified payment destinations.
11. Employee-by-AZM-ID enrollment must work end-to-end.
12. Contact-discovery/chat relationship needs product-level authorization/privacy semantics distinct from payment authority.
13. EWA fee policy must become admin-configurable.
14. Generalized payout-plan engine must replace Smart Route special cases.
15. QR payment intent must be designed and integrated with canonical payment/receipt semantics.
16. Currency/loyalty/USDC precision and Decimal-vs-Number boundaries need a systematic authority audit.
17. Business payout/payroll source-of-funds accounting needs full ledger/reconciliation treatment.
18. Moolre-only provider abstraction should replace/deprecate provider naming leakage in current fiat code.
19. Consumer Flutter API compatibility coverage must become a real CI gate once the mobile API token/contract decision is finalized.

---

## 16. Product journey target map

| Journey | Canonical target |
|---|---|
| User → user payment | AZM-ID resolution → recipient preview → explicit confirmation → internal ledger settlement |
| User → business payment | BIZ-ID resolution → business preview → explicit confirmation → business financial participant settlement |
| Local fiat deposit | Moolre collection → provider evidence → exact-once settlement → USDC liability |
| Local fiat withdrawal | Moolre execution → provider evidence → reconciliation → authoritative settlement |
| Crypto withdrawal | custody execution → provider/chain evidence → reconciliation → authoritative settlement |
| QR payment | signed short-lived payment intent → scan → live resolve → confirm → settle → receipt |
| Phone contact discovery | hashed verified-phone matching → privacy controls → chat/contact graph |
| Employee enrollment | AZM-ID resolve → preview → business confirms association |
| Payroll | compensation authority → business source-of-funds → employee monetary liability → receipt/history |
| EWA | wage eligibility → advance reservation → employee monetary credit → employer settlement |
| Scheduled allowance | generalized PaymentPlan → payout batch/item → internal-user settlement |
| Scheduled business payout | PaymentPlan + recipient/rule snapshot → payout batch → itemized execution |
| Scheduled local-fiat withdrawal | PaymentPlan → Moolre destination → payout batch/item → provider reconciliation |

---

## 17. Engineering rules for future work

1. Do not create parallel ledgers or parallel financial mutation systems.
2. Do not make Smart Route a permanent collection of special cases.
3. Do not treat AZM loyalty points as USDC or other customer cash.
4. Do not make MTN/Vodafone/Telecel separate AZAMAN fiat providers unless an actual separate contract exists.
5. Do not use friendship state as the financial destination identity.
6. Do not make numeric database user IDs the customer-facing payment primitive.
7. Do not make QR tokens bearer authorization.
8. Do not let employee payroll/EWA code invent its own balance authority.
9. Do not rely on the happy path; every financial mutation must be tested under duplicate, retry, concurrent and ambiguous outcomes.
10. Every architecture discovery that changes the product contract must be recorded in Planning immediately.
11. Every implementation must be re-audited repo-wide after merge so stale consumers, duplicate paths and terminology leakage are found.
12. A green test suite does not prove architectural correctness by itself.

---

## 18. Implementation sequencing

The current architecture work should continue in this broad order, while allowing independent research in parallel:

### Phase A — close already-identified P0s

- Smart Route atomic execution and canonical provider/withdrawal settlement.
- Admin reject safety.
- Vault release claim.
- Savings withdrawal claim.
- Order Book runtime/concurrency integrity.
- Resolve or formally retire stale Shared Vault.

### Phase B — financial semantics

- Payroll/EWA migration from AZM loyalty to authoritative monetary liability.
- Business source-of-funds and EWA employer obligation accounting.
- Admin-configurable EWA policy.
- Employee AZM-ID enrollment.
- Business payout destination semantics.

### Phase C — identity/payment primitives

- Unified identity resolution.
- AZM-ID/BIZ-ID first-class payment destinations.
- Recipient preview and identity snapshot receipts.
- Internal transfer convergence.

### Phase D — generalized scheduled payments

- Design canonical PaymentPlan/recipient/rule/batch/item semantics.
- Migrate Smart Route into compatibility façade.
- Add allowance, internal recurring payment, Moolre recurring withdrawal and business payout use cases.
- Add role/tag/rank grouping, overrides, bonuses and notes.
- Add approval policy and execution snapshots.

### Phase E — payment requests

- QR payment intent.
- QR scan/preview/confirm/settle.
- Reusable payment-intent primitives for future tap/NFC flows.

### Phase F — cross-repo release readiness

- Flutter ↔ backend contract gate.
- Business Portal ↔ backend contract gate.
- Admin data contracts.
- Realtime reconciliation.
- Observability/recovery/load/red-team proof.
- Production activation gates.

The phases are intentionally architectural, not promises of one giant PR. Each implementation must be split at the smallest coherent authority boundary and proven before merge.

---

## 19. Decision log

### DEC-2026-09-20-01 — Moolre is the current fiat provider
Decision: treat Moolre as the only current external fiat provider contract.

Reason: there is no direct AZAMAN MTN API contract; Moolre already abstracts supported Ghanaian network destinations.

### DEC-2026-09-20-02 — AZM-ID/BIZ-ID are payment primitives
Decision: customer-facing identity/payment flows must use stable AZM-ID/BIZ-ID rather than raw numeric User IDs.

Reason: identity must be explicit, human-facing and confirmable before money movement.

### DEC-2026-09-20-03 — Smart Routes become generalized scheduled payments
Decision: recurring payments become a generalized plan/batch/item engine.

Reason: product requirements extend beyond withdrawals and savings to allowances, internal payments, Moolre withdrawals, payroll and grouped business payouts.

### DEC-2026-09-20-04 — AZM loyalty is separate from money
Decision: `azmBalance` is loyalty, not cash.

Reason: schema semantics already separate it from monetary `availableBalance`; payroll/EWA code currently violates that contract.

### DEC-2026-09-20-05 — QR is a request, not authorization
Decision: QR/tap flows must resolve a short-lived payment intent and require explicit payer confirmation.

Reason: a visual/bearer token must never directly authorize a financial mutation.

### DEC-2026-09-20-06 — Contact discovery is separate from payment authorization
Decision: known-phone contact relationships may simplify chat/social access, but cannot authorize financial movement.

Reason: social convenience and financial authorization have different security/privacy requirements.

### DEC-2026-09-20-07 — Business employee enrollment is AZM-ID based
Decision: business owners enroll employees by AZM-ID with identity preview.

Reason: AZM-ID is the intended human-facing user identity and the portal already presents this product contract.

---

## 20. Session-continuity requirement

At the start of every future AZAMAN engineering session:

1. Read this document.
2. Read `ROADMAP.md`.
3. Read `ARCHITECTURE.md`.
4. Read `CURRENT_STATE.md`.
5. Check the current main branch and open PRs for the affected repository.
6. Re-verify any "current" claim against live code before implementing.
7. Preserve the decisions in this document unless new evidence proves a decision incorrect.
8. When new product requirements are discovered, update Planning before they can be forgotten.

**This document is the durable record of the product-direction context captured on 2026-09-20.**
