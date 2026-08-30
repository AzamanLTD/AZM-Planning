# AZAMAN Master Roadmap — 2026-08-30

**Status:** ACTIVE / PRODUCTION HARDENING + COMPETITION DELIVERY
**Purpose:** Keep the whole platform moving as one system. This document records the current architectural truth, completed foundations, open risks, and the next coherent implementation batches.

> This roadmap supersedes stale point-in-time handoffs for planning purposes. Historical documents remain useful as design context; repository state and verified CI remain the evidence of implementation.

## 1. Product target

AZAMAN is not a collection of disconnected apps. The target is a personal operating system for **money + commerce + work + communication**.

The platform backbone is:

**Identity & Trust → Unified Money/Ledger → Domain Verticals → SDUI/Presentation → Realtime/Eventing → Social Graph → Notifications → Observability/Reconciliation**

Repository responsibilities:

| Layer | Repository | Responsibility |
|---|---|---|
| Experience | `AZM-frontend` | Consumer/mobile experience |
| Source of truth | `AZM-backend` | Identity, money, domain state, APIs, events |
| Business control plane | `AZM-businessPortal` | Merchant operations + storefront control |
| Administrative control plane | `AZM-adminPortal` | Governance, risk, workforce, oversight |
| Engineering memory | `AZM-Planning` | Architecture, roadmap, verification history |

## 2. Current verified baseline

### COMPLETE / VERIFIED

- Retail escrow-capable checkout architecture hardened across backend + Flutter.
- Checkout operation identity/idempotency and typed failure/retry semantics.
- Inventory reservation and atomic checkout protections.
- Monotonic escrow/order transitions and double-settlement protections.
- Storefront commerce convergence across backend/frontend/business portal.
- Business storefront optimistic concurrency with conflict recovery.
- Server-side transaction quote/FX persistence and atomic consumption.
- Mobile auth refresh-token revocation and production backend connection.
- Android cleartext-traffic hardening.
- Frontend startup/performance and CI quality gates.
- Backend CI with merge-queue support, timeout and read-only workflow permissions.
- Frontend CI with Flutter quality/build gates and merge-queue support.
- Admin portal CI/build validation.
- Business portal production-build validation workflow.
- Durable staff/control-plane foundation, workforce/presence/audit/dispute/governance surfaces identified and integrated into the administrative direction.
- Canonical Nitro tier policy centralized in backend.
- AZM Planning repository established as the central engineering memory.

### IMPORTANT: COMPLETE DOES NOT MEAN PRODUCT-COMPLETE

The above is a verified foundation. It does **not** mean every vertical, workforce workflow, financial governance workflow, or competition-facing experience is complete.

## 3. Master workstreams

### WS-A — Financial truth and trust layer
**Priority: P0**

Goal: every money movement has one authoritative lifecycle, ledger effect, authorization boundary, idempotency behavior, audit trail and reconciliation path.

Required work:

- unify financial operation identity across direct payment, escrow, withdrawal, refund and future financial products;
- audit every balance mutation for transactionality and double-spend resistance;
- verify escrow fund/release/refund/dispute invariants against ledger reconciliation;
- make admin financial actions explicitly authorized, audited and replay-safe;
- build reconciliation/exception reporting rather than relying only on transaction rows;
- ensure domain events are emitted only after authoritative state commits;
- establish correlation/request IDs across financial operations.

Exit evidence: invariant tests, concurrency tests, audit coverage, reconciliation checks and end-to-end happy/error paths.

### WS-B — Retail / commerce vertical
**Priority: P0 — competition-facing**

Goal: make retail a polished reference vertical and reusable commerce primitive.

Required work:

- complete checkout happy path with direct + optional escrow;
- make inventory, pricing, variant and order snapshots authoritative;
- strengthen payment recovery for ambiguous network failures;
- build customer order tracking and business order operations end-to-end;
- finish storefront specialization as a real retail collection/shelf experience;
- ensure business portal and consumer app expose the same canonical order states;
- test concurrency: duplicate taps, retry-after-timeout, stock races, conflicting storefront edits.

### WS-C — Marketplace vertical expansion
**Priority: P1**

Reuse shared commerce primitives for:

- Restaurant — menu/dining-room experience;
- Hotel — property/room explorer and booking;
- Transit — journey/departure-board and seat reservation;
- later commerce verticals.

Do not clone retail pages. Each vertical must have structural identity while reusing identity, money, order, notification and realtime primitives.

### WS-D — Escrow + ticket workspace
**Priority: P0/P1**

Escrow is a signature AZAMAN trust feature. Target lifecycle:

`DRAFT → FUNDED → IN_PROGRESS → PENDING_SETTLEMENT → SETTLED`

with dispute/admin paths:

`DISPUTED → ADMIN_REVIEW → RELEASED / REFUNDED`

Required experience:

- create from conversation;
- explicit amount/fee disclosure;
- fund with idempotency;
- shared ticket workspace;
- delivery submission;
- buyer approval/request-changes;
- evidence-backed dispute;
- clear locked/pending/settled money states;
- realtime activity + notifications.

Backend remains authoritative; UI never invents financial state.

### WS-E — Administrative control plane
**Priority: P0/P1**

The CEO/global super-admin should ultimately see the whole platform while delegated staff see/do only what their authority permits.

Required progression:

`RBAC → durable staff → lifecycle/presence → departments → work queues → assignments → audit/event stream → financial governance → unified disputes → CEO command center → workforce UI`

Workforce states must become explicit:

`INVITED / ACTIVE / AWAY / SUSPENDED / INACTIVE / TERMINATED`

Operational access must respond immediately to suspension/inactivation.

### WS-F — Business operating system
**Priority: P1**

Business Portal must become the merchant operating layer rather than a collection of screens.

Converge:

- storefront;
- products/inventory;
- orders;
- reservations;
- finance;
- employees/scheduling/payroll;
- restaurant operations;
- hotel operations;
- transit operations;
- messages/notifications;
- analytics.

Every important action needs backend authority and clear failure/retry semantics.

### WS-G — Consumer experience / UX
**Priority: P0/P1**

The mobile app is the product experience. Preserve strong existing foundations: Riverpod, GoRouter, Socket.IO, secure storage, notifications, Sentry, SDUI and existing specialized experiences.

Do not introduce competing state-management or animation philosophies.

Focus on:

- fast startup;
- coherent navigation;
- category-specialized marketplace experiences;
- trustworthy money/checkout states;
- offline/ambiguous-network recovery;
- delightful but purposeful motion;
- accessible error and loading states.

### WS-H — Social / communication graph
**Priority: P1/P2**

Unify existing messaging, profiles, follows/social relationships, activity and notifications instead of creating parallel systems for each vertical.

The graph should eventually connect:

`identity → people → businesses → transactions → tickets → activity → notifications`

without leaking private financial data.

### WS-I — Observability / reconciliation / operations
**Priority: P0**

Production readiness requires visibility into failures, not only logs.

Build:

- structured correlation IDs;
- domain event observability;
- financial reconciliation jobs;
- stuck-operation detection;
- expired escrow/order/booking detection;
- notification delivery failure tracking;
- admin operational queues;
- meaningful health/readiness checks.

## 4. Cross-system invariants

These are platform rules, not UI suggestions.

### Money

- available balance only changes through authoritative financial services;
- escrow fund: `available -= principal + fee`, `escrowLocked += principal`, `systemFee += fee`;
- release: `escrowLocked -= principal`, `payeeAvailable += principal`;
- refund: `escrowLocked/dispute -= principal`, `payerAvailable += principal`;
- every operation is idempotent where retries are possible;
- concurrent calls cannot cause double payout or double reservation.

### Commerce

- inventory cannot become negative through concurrent checkout;
- order snapshots preserve the purchased variant/price context;
- customer and business see the same canonical order lifecycle;
- status transitions are monotonic and authority-controlled.

### Booking

- same room + overlapping confirmed dates → at most one confirmed reservation;
- same trip + same seat → at most one confirmed reservation;
- seat/room holds expire safely and are idempotent.

### EWA / payroll

- outstanding EWA cannot exceed policy limit;
- repayment amount is server-authoritative;
- payroll net cannot become negative;
- consent/disclosure version is stored server-side.

### Administration

- CEO/global authority is explicit;
- delegated authority is scoped;
- staff suspension/inactivation affects operational access;
- sensitive admin/financial actions are audited;
- audit events never contain secrets or unnecessary sensitive data.

## 5. Competition delivery sequence

### Phase 1 — Trustworthy core

- financial invariants;
- checkout + escrow;
- order tracking;
- storefront concurrency;
- operational observability;
- CI/release gates.

### Phase 2 — Demonstrable AZAMAN experience

- polished retail journey;
- signature escrow/ticket shared workspace;
- business storefront editor;
- real-time activity/notifications;
- strong demo data and deterministic demo paths.

### Phase 3 — Vertical breadth

- hotel;
- restaurant;
- transit;
- employee/EWA.

### Phase 4 — Platform intelligence

- social graph;
- cross-vertical recommendations/discovery;
- executive command center;
- workforce automation;
- reconciliation and operational intelligence.

## 6. Definition of done for substantial batches

A batch is not done merely because code exists.

Before declaring completion, verify:

1. architecture/callers understood;
2. schema/migrations checked;
3. authorization boundary checked;
4. idempotency/concurrency behavior checked where relevant;
5. domain events/notifications checked;
6. tests added or strengthened;
7. exact CI head verified;
8. integration path verified where practical;
9. AZM-Planning updated with status/evidence/risk;
10. stale/superseded PRs and branches handled deliberately.

Do not split a coherent feature into cosmetic micro-PRs solely to obtain green checks. Prefer meaningful batches with complete verification.

## 7. Current risks to burn down

- Admin Portal has substantial pre-existing type-system debt; do not hide it by weakening type checking.
- Business Portal CI infrastructure must be proven operational, not merely committed to the repository.
- E2E cleanup warnings must be eliminated so test infrastructure itself is trustworthy.
- Financial reconciliation needs to mature from transaction-row inspection into explicit invariant/exception detection.
- Cross-repo contracts need continued automated coverage so backend changes cannot silently break Flutter or portal behavior.
- Older handoff documents contain historical state and must not override current repository evidence.

## 8. Progress ledger

| Date | Batch | Status | Evidence |
|---|---|---|---|
| 2026-08-28 | Retail escrow capability/enforcement | VERIFIED | Backend + Flutter + tests |
| 2026-08-29 | Checkout operation identity/integrity | VERIFIED | Backend/Flutter CI |
| 2026-08-29 | Storefront commerce convergence | VERIFIED | Backend/Flutter/Business Portal |
| 2026-08-29 | Transaction quote engine | VERIFIED | Backend implementation/tests |
| 2026-08-29 | Control-plane foundation | VERIFIED | Backend/Admin Portal |
| 2026-08-29 | Auth/security hardening | VERIFIED | Backend/Flutter/Business Portal |
| 2026-08-30 | Checkout failure semantics | VERIFIED | Flutter CI |
| 2026-08-30 | Canonical Nitro policy | VERIFIED | Backend CI |
| 2026-08-30 | Cross-repo CI hardening | VERIFIED | Repository-specific gates |
| 2026-08-30 | Central planning brain | VERIFIED | `AZM-Planning` |

## 9. Operating rule for future agents

**Do not beat around the bush.** Start from the master architecture, inspect real repository state, select the highest-value incomplete workstream, implement a coherent batch, verify it, update this planning repository, and continue.

When blocked by tooling, use another available route to inspect/verify the actual state. When a limitation prevents an operation, record the limitation and maximize the work that can still be completed safely. Never manufacture completion.
