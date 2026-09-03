# AZAMAN Total Engineering Roadmap

**Priority authority:** YES  
**Status:** ACTIVE  
**Last reconciled:** 2026-09-03 UTC

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
