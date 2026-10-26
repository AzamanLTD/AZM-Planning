# AZAMAN Total Engineering Roadmap

**Priority authority:** YES  
**Status:** ACTIVE  
**Last full roadmap reconciliation:** 2026-09-03 UTC; **multi-asset money architecture scope added:** 2026-10-26 UTC. The rest of the roadmap has not been globally re-audited by this change.

This is the **single active execution plan** for the five-repository AZAMAN platform. It incorporates the prior master roadmap and Lyra's independent assessment. Dated plans are historical evidence, not competing instructions.

## North-star definition of done

AZAMAN is ready only when every important journey has one authoritative source of truth; every mutation is explicitly authorized, correctly tenant-scoped, atomic where required, idempotent/retry-safe, auditable and observable; state transitions are enforced; cross-repo contracts are verified; clients converge after realtime/network failures; deployment/rollback/migration/secrets are proven; critical admin operations are tested; and production behavior has been exercised under concurrency/load and adversarial conditions.

## Priority model

**P0:** financial correctness, tenant isolation, state-machine integrity, cross-repo contract correctness, deployment/rollback/migration/secrets, critical Admin mutation safety.  
**P1:** concurrency/idempotency, realtime reconciliation, Business OS, Experience Blueprint, customer authority, broader contract coverage.  
**P2:** dead-code/duplication cleanup, test architecture, component extraction.  
**P3:** measured performance and UX polish.

---

# WAVE 0 — CONTROL THE ENGINEERING BASELINE

**Goal:** eliminate ambiguity before more implementation.

- [ ] Reconcile all open backend PRs against current `main`; identify canonical vs duplicate/stale attempts.
- [ ] Close/supersede duplicate PRs rather than carrying parallel fixes.
- [ ] Delete stale branches only after confirming they contain no unique required work; branch cleanup is part of completion.
- [ ] Repeat branch audit across Admin, Business Portal and Flutter.
- [x] Establish Planning repo as the only active navigation surface — completed in PR #24, merged at `da489d71fbe76f411494315c7e20f2c85d1bbc4b`, with live state subsequently recorded on `main`.
- [x] Keep historical session material out of the active surface; superseded root journals were removed while exact historical content remains recoverable from Git history.
- [x] Record exact baseline state and next actions in `CURRENT_STATE.md`.
- [ ] Verify deployment environments, configuration boundaries and current release mechanism rather than assuming they exist.

**Exit:** active plan is obvious within minutes of entering the repo; remaining Wave 0 work is repository-operations/deployment verification.

# WAVE 1 — FINANCIAL + CONCURRENCY CORRECTNESS

**Goal:** prove that money and high-value state cannot be duplicated, lost or authorized incorrectly.

## Backend financial surfaces

- [ ] Wallet: deposit, withdraw, approve/reject, freeze/release; verify atomic balance changes and idempotency.
- [ ] Escrow: fund, settle, release, refund, dispute, expiry; verify ledger/balance invariants and concurrent races.
- [ ] Trades: open, fund, release, force-release, cancel, dispute; verify duplicate/race safety.
- [ ] EWA: eligibility, request, approval, repayment/settlement; verify business scope and retry behavior.
- [ ] Payroll: process and disbursement; verify PR #131 and all financial side effects.
- [ ] Orders/invoices: payment, settlement, refund/void and monetary transitions.
- [ ] Reservations: payment, cancellation and refund.
- [ ] Dine-in: `confirmAndPay`, invoice creation/payment/closure and adapter/transaction semantics.

## Concurrency methodology

For every mutation, explicitly inspect **read → decision → write**. Replace unsafe check-then-write patterns with conditional writes, transactions, unique constraints or deterministic state claims as appropriate.

Required race cases include:

- [ ] wallet balance mutation races;
- [ ] escrow hold/settle/release races;
- [ ] trade funding races;
- [ ] notification dedup/create races;
- [ ] order create/confirm races;
- [ ] reservation/seat/room races;
- [ ] payroll/EWA serializable conflicts;
- [ ] dine-in finalize/payment race;
- [ ] concurrent admin approvals/releases.

**Exit:** invariant tests + DB-backed concurrency tests + exact-head CI for every changed repository.

# WAVE 2 — TENANT, ACTOR AND STATE-MACHINE INTEGRITY

## Tenant boundary

Complete an object-by-object matrix for invoices, orders, reservations, locations, products, staff permissions, reports, notifications and remaining Business OS resources. Every mutation must establish business scope from trusted request context and/or prove the target belongs to that scope.

**Principle:** a globally unique ID is an identifier, not authorization.

## State machines

For employee lifecycle, shifts, swaps, time off, payroll, EWA/withdrawals, orders, reservations and relevant admin operations:

- [ ] enumerate legal transitions;
- [ ] reject illegal transitions server-side;
- [ ] ensure side effects occur only on the winning transition;
- [ ] protect terminal states from replay;
- [ ] make concurrent losers return canonical committed state where appropriate;
- [ ] encode every important transition in regression tests.

Immediate known boundary: `ShiftService.updateShift()` must not accept generic `status` mutation; state-changing attendance actions remain dedicated transitions.

Also audit route-level permission parity, including shift rotation creation and EWA/attendance endpoints.

# WAVE 3 — CRITICAL ADMIN TEST DEBT

**Goal:** make the most dangerous control-plane mutations executable and trustworthy before release.

Admin Portal is currently the weakest surface because it has only a small number of tests relative to its financial/governance responsibility.

Priority tests:

- [ ] withdrawals: approve/reject/freeze/settle;
- [ ] escrow disputes: force-cancel/force-release;
- [ ] fee engine: fee application/profile changes;
- [ ] War Room: privileged actions/dispute resolution;
- [ ] dashboard: KPI contract accuracy;
- [ ] verify every financial mutation routes through the dedicated financial API facade;
- [ ] test optimistic mutation removal + rollback semantics rather than trusting refetch alone.

Do not add frontend test dependencies unless the repo's typecheck/test configuration supports them. Never weaken type checking to accommodate tests.

# WAVE 4 — CROSS-REPO AUTHORITY + EXPERIENCE TRUTH

## Dine-in first

Audit the four-surface contract:

`Flutter → Backend → Business Portal → Admin visibility`

Verify:

- request field names and response shapes;
- `tabId`, product/selection/quantity semantics;
- finalize → confirmAndPay ordering;
- transaction boundary and adapter I/O;
- invoice/payment/closure authority;
- socket events: `dine_in_tab_opened`, `dine_in_item_added`, `dine_in_tab_finalized`, `dine_in_tab_paid`;
- Flutter consumers and reconnect behavior;
- concurrent finalize/payment behavior;
- canonical recovery after ambiguous network failure.

## Experience Blueprint

Treat Blueprint as a first-class cross-repo contract:

**Business Portal configures → Backend validates/stores/serves authoritative configuration → Flutter renders it.**

No client may silently invent configuration or maintain a competing interpretation.

## Remaining journeys

- [ ] withdrawal: Flutter → Backend → Admin → Flutter;
- [ ] marketplace: discovery → detail → cart → checkout → order;
- [ ] hotel: inventory → room state → booking → amount → success/check-in;
- [ ] transit: route → server fare → booking → result;
- [ ] restaurant/dine-in;
- [ ] auth/session lifecycle;
- [ ] notifications/unread/action routing.

# WAVE 5 — REALTIME + API CONTRACT CONVERGENCE

- [ ] Build producer → transport → payload → consumer → fallback matrix for important events.
- [ ] Events are emitted only after authoritative commits.
- [ ] Realtime is a convergence signal, never financial truth.
- [ ] Add a reconciliation/refetch mechanism for missed socket events and reconnects.
- [ ] Ensure idempotent query invalidation/refresh rather than blind local mutation.
- [ ] Audit API producer/consumer contracts for stats, dashboards, reservations, bookings, hotel rooms, transit, notifications, employees, payroll, invoices and products.
- [ ] Add executable cross-repo contract tests where practical.

# WAVE 6 — BUSINESS PORTAL + MARKETPLACE

Business Portal:

- [ ] audit every KPI for authoritative `/stats` or equivalent backend contracts;
- [ ] eliminate accidental `.slice()`, fixed-limit truncation, client aggregation and fake/static fallback from authoritative metrics;
- [ ] inspect Finance, Invoices, Products and Reservations for hidden business calculations;
- [ ] extract oversized components only after behavior is locked by tests;
- [ ] verify Experience Studio simulator and published storefront are behaviorally consistent.

Marketplace:

- [ ] runtime-aware reachability audit;
- [ ] remove only proven dead code;
- [ ] identify duplicate API helpers and hidden response transformations;
- [ ] preserve category-native structural identity while reusing canonical commerce primitives.

# WAVE 7 — PRODUCTION OPERATIONS

These are P0 release requirements, not late polish.

- [ ] prove a staging/non-dev deployment path;
- [ ] prove production configuration and environment separation;
- [ ] document secret storage, rotation and incident response for JWT/DB/provider/FCM/MoMo credentials;
- [ ] prove rollback of code and database changes;
- [ ] define backward-compatible Prisma migration discipline: expand → migrate/backfill → contract;
- [ ] verify populated-table destructive-change strategy;
- [ ] structured logs + correlation/request IDs;
- [ ] error-rate and latency monitoring;
- [ ] financial anomaly/reconciliation alerts;
- [ ] worker crash/restart recovery;
- [ ] stuck-operation and missed-event detection.

# WAVE 8 — LOAD + SECURITY RED TEAM

Minimum representative exercises:

- [ ] 100 concurrent checkouts;
- [ ] 10 concurrent admin withdrawal approvals/releases;
- [ ] 500 concurrent socket connections;
- [ ] retry storms/ambiguous provider responses;
- [ ] tenant IDOR attempts;
- [ ] privilege escalation and delegated-permission abuse;
- [ ] replay/idempotency attacks;
- [ ] state manipulation through generic PATCH/API abuse;
- [ ] financial double-spend/double-settlement attempts;
- [ ] sensitive-data leakage in audit/events/errors.

# WAVE 9 — UX, ACCESSIBILITY, PERFORMANCE

Only after correctness is secure:

- [ ] loading/empty/error/success states for every journey;
- [ ] screen-reader semantics/focus management;
- [ ] reduced motion;
- [ ] touch/keyboard target sizing;
- [ ] destructive-action confirmation;
- [ ] optimistic rollback semantics;
- [ ] Flutter rebuild/render hotspots;
- [ ] unnecessary API calls and payloads;
- [ ] Business Portal monolith extraction;
- [ ] measure before optimizing.

# WAVE 10 — RELEASE REHEARSAL

- [ ] execute `RELEASE_CHECKLIST.md` from a clean baseline;
- [ ] run full cross-repo E2E journeys;
- [ ] verify migration and rollback rehearsal;
- [ ] verify observability/alerts;
- [ ] verify branch/PR hygiene;
- [ ] verify exact-head CI for all changed repos;
- [ ] independently review final risks (Jarvis + Lyra-style adversarial assessment);
- [ ] cross-repo sign-off only after evidence exists.

---

## Parallelization rule

Independent workstreams may proceed in parallel, but never parallelize two changes that touch the same logical authority boundary without first choosing one canonical implementation. While CI runs, perform independent research/audits rather than waiting idle.

## Completion rule

A checkbox is checked only with evidence. Every completed item must record repository, commit/PR, test/CI evidence and residual risk in `CURRENT_STATE.md`.


---

# WAVE 11 — PRODUCT-INTENT AND FINANCIAL-DOMAIN CONVERGENCE (2026-09-20)

**Canonical contract:** `docs/PRODUCT_INTENT_AND_TARGET_ARCHITECTURE_2026-09-20.md`

This wave captures product requirements that must constrain future implementation. It is not a feature-shopping list; the financial/identity semantics are part of the architecture.

## Moolre-only current fiat rail

- [ ] Treat Moolre as the only current external fiat provider contract.
- [ ] Remove/deprecate direct-provider terminology that implies AZAMAN has separate MTN/Telecel/Vodafone provider contracts.
- [ ] Keep local network/operator selection as Moolre destination information.
- [ ] Audit all fiat code paths for provider abstraction leakage and canonicalize them behind the Moolre rail.

## Money vs AZM loyalty

- [ ] Prove every payroll/EWA financial credit targets monetary liability, never `User.azmBalance`.
- [ ] Keep AZM loyalty accounting isolated from USDC/customer money.
- [ ] Reconcile employee compensation source-of-funds and obligations through the authoritative ledger.
- [ ] Make EWA fee/limits policy admin-configurable.

## Identity and recipient authority

- [ ] Establish AZM-ID as first-class user payment identity.
- [ ] Establish BIZ-ID as first-class business payment identity.
- [ ] Build canonical identity resolution + recipient preview.
- [ ] Preserve recipient identity snapshots on receipts/audit records.
- [ ] Fix employee enrollment so Business Portal AZM-ID input resolves to the canonical user before association.
- [ ] Model AZM-ID → BIZ-ID payments as business-participant settlement, not blind owner-user credit.

## Social/contact graph

- [ ] Evolve phone discovery into a privacy-safe known-contact relationship.
- [ ] Allow chat convenience without making a friend request mandatory where policy permits.
- [ ] Preserve block, opt-out and privacy controls.
- [ ] Keep social/contact access distinct from financial authorization.

## Generalized scheduled payments

- [ ] Design `PaymentPlan` / target / recipient / rule / override / batch / item semantics.
- [ ] Support recurring AZM-ID transfers, allowances, Moolre withdrawals and business payouts.
- [ ] Support role/tag/rank groups, per-recipient amounts, bonuses, notes and execution snapshots.
- [ ] Add single-winner occurrence claims, idempotency, concurrency safety and reconciliation.
- [ ] Migrate Smart Routes behind the generalized scheduler instead of adding more special cases.
- [ ] Audit current Smart Route race/reconciliation defects before migration.

## QR / payment intent

- [ ] Introduce short-lived signed payment requests.
- [ ] Scan must resolve current recipient and amount before payer confirmation.
- [ ] QR must never directly authorize money movement.
- [ ] Produce canonical receipts.
- [ ] Reuse the payment-intent primitive for future NFC/tap-to-pay flows.

## Business employee / payroll / EWA

- [ ] Business Portal adds workers by AZM-ID with identity confirmation.
- [ ] Flutter worker experience consumes authoritative employee/payroll/EWA data.
- [ ] Payroll uses real monetary accounting and business source-of-funds.
- [ ] EWA advances are monetary obligations/credits, not loyalty minting.
- [ ] Generalized payout scheduling must not duplicate payroll settlement logic.

**Exit:** these requirements are either implemented behind canonical authority boundaries or explicitly marked unresolved with evidence in `CURRENT_STATE.md`.



---

# WAVE 12 — MULTI-ASSET MONEY, PRODUCT DENOMINATION AND SETTLEMENT (2026-10-26)

**Priority:** P0 financial correctness  
**Canonical architecture:** [docs/MULTI_ASSET_MONEY_AND_SETTLEMENT_ARCHITECTURE_2026-10-26.md](docs/MULTI_ASSET_MONEY_AND_SETTLEMENT_ARCHITECTURE_2026-10-26.md)  
**Status:** PLANNED — design decision recorded; no application behavior or live-money route is activated by this documentation.

AZAMAN must distinguish currency/asset held, product obligation denomination, display valuation, executable quote, funding plan, customer liability and evidenced settlement. Domestic obligations default to GHS; the source asset never silently changes the contract. Use the existing authoritative ledger, explicit ASSET_CONVERSION boundary, durable operation/idempotency machinery and route-bound quote authorities.

## Decisions that constrain every feature

- [ ] New domestic Susu, savings/vault contracts and local merchant prices default to GHS denomination only when real GHS funding/settlement is backed and reconciled. Explicit USDC-denominated products remain possible with clearly estimated GHS display values.
- [ ] Susu denomination is explicitly selected and contract-versioned before activation, then immutable. Reject dynamic 50/50 or 60% currency-majority switching; any composition-based suggestion uses only declared/consented preferences and never changes a running contract.
- [ ] Use one Funding Planner over actual spendable asset buckets. GHS first by policy; when short, offer a quote-backed USDC conversion. Display valuations never authorize debits.
- [ ] Distinguish “send this asset amount” from “recipient receives this currency amount.” Recipient settlement preference may preselect UI but cannot silently alter sender intent.
- [ ] Use reusable ≈ / estimate treatment across wallet, Susu, savings, vault, transfer and checkout; show unavailable rather than a fabricated zero. Quote estimates include known costs and are not represented as guarantees.
- [ ] Conversion fees and AZAMAN margin are visible before consent and recognized at supported settlement, not continuously skimmed from customer pools or inferred from revaluation.
- [ ] Prefer quote-backed on-demand asset sourcing for new conversions. Customer-owned USDC, platform-owned inventory, provider-held assets, clearing and customer liabilities must remain explicitly distinct and reconciled.
- [ ] FX execution gates are route-specific. A conversion/GHS-valued obligation requires valid executable quote authority; a pure same-asset transfer does not automatically depend on a stale unrelated USDC/GHS display rate.
- [ ] Susu payout requires fully funded contract math, a single-winner payout identity and recoverable provider ambiguity. Default to no platform advance and no partial payout; any exception requires a separately funded and approved risk product.
- [ ] Do not reinterpret existing USDC-labelled rows as GHS. Audit contract versions and migrate only through a controlled, consent-aware, ledger-reconciled plan.

## Ordered implementation waves

1. **Phase 0 — Read-only audit:** inventory all financial routes, product amount fields, provider rails, quote authority, ledger legs, operations, state machines and reconciliation; verify the current native-GHS customer path versus existing GHS liquidity/off-ramp authority; audit legacy Susu/vault contract strata; get current provider/legal confirmation.
2. **Phase 1 — Contract freeze:** versioned typed currency/asset/amount/quote/settlement API contracts and cross-repo tests. Reuse existing models where possible; no invented endpoints or duplicate quote/ledger.
3. **Phase 2 — Asset/settlement backbone:** prove real GHS customer-liability funding, spend, withdrawal/reversal and reconciliation; complete the existing asset-conversion/inventory economics boundary; align route-specific FX gates.
4. **Phase 3 — Funding Planner and quote execution:** reserve source funds, require quote-bound consent, execute with durable operation identity, recover ambiguous provider outcomes and enforce standing-instruction caps.
5. **Phase 4 — Susu:** immutable denomination/contract version, correct contribution and payout unit, mixed-asset funding, default/late-provider semantics, one-winner full-pot settlement.
6. **Phase 5 — Savings/vault:** separate progress-only targets from funded/locked money; denomination-aware deposit, auto-rule, break, maturity and release.
7. **Phase 6 — Marketplace/restaurant/dine-in:** domestic GHS order and merchant-receivable contract with a real matching settlement route, payment/refund/reconciliation proofs.
8. **Phase 7 — Peer/remittance:** explicit source-asset versus recipient-currency intent, preference semantics, sender-disclosed conversion costs and future corridor abstraction.
9. **Phase 8 — Client/portal convergence:** shared money models, shared estimate display, funding-source selection, quote confirmation, pending continuation/recovery, accessibility and exact API compatibility.
10. **Phase 9 — Controls/release:** multi-asset reconciliation, anomaly alerts, provider runbooks, treasury limits, legal/compliance sign-off, migration/rollback rehearsal and independent red-team review.

## Required gates

- [ ] Backend route/field/asset matrix is verified against current main before implementation.
- [ ] #367 native-GHS balance/valuation contract is not treated as proof of live GHS customer funding.
- [ ] #368 universal FX gate is reviewed against the route-specific policy before it becomes final.
- [ ] Existing #369 Susu payout safety is preserved, but its currency contract is separately audited against the product denomination decision.
- [ ] Frontend #203 unknown/malformed-value behavior is preserved; client code never computes authoritative transaction pricing.
- [ ] Critical ledger/concurrency/replay tests use disposable real PostgreSQL, not production Neon; exact-head CI and final source/diff audit are required.
- [ ] No production activation, rollout or financial data repair proceeds without its separate explicit authorization.

**Exit:** every supported product states the exact obligation currency, eligible funding assets, quote/fee authority, ledger postings, provider finality, failure/reversal behavior and reconciliation proof. Green CI alone is insufficient.