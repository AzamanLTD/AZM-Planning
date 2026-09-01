# AZM Hardening Closeout — 2026-09-01

## Purpose

Record the verified closeout state of the pre-UI hardening phase. This document records implementation and verification evidence; it does not treat planned discussion as completed work.

## Status

**State: IN PROGRESS until the post-merge Backend `main` PostgreSQL gate for correlation IDs completes successfully.**

The correlation implementation was merged as Backend PR #82 at `8323b0cfec1f41cf68c31165d6d38ac6fadf25f2` after PR head `24d750090fad5acda6f963d40311441fa62f0c0e` passed the full PostgreSQL-backed test workflow and database backup/restore drill.

The post-merge `main` run is independently required because merge validation and branch validation are separate evidence.

## Completed hardening slices

### Financial realtime contracts

Executable Backend contract coverage now verifies the high-value financial event boundaries established during the hardening sequence, including:

- `escrow_funded` and `escrow_settled`, including timestamp fields from committed escrow state;
- `escrow_refunded` expiry semantics;
- `escrow_pending_settlement` commit ordering and settlement-wins behavior;
- `withdrawal_progress` / `withdrawal_settled` user topology;
- separate `admin_alert` withdrawal topology targeting `admin_spy`;
- `invoice_paid` room/payload/post-commit semantics;
- `biz_notification` producer/consumer shape;
- `order.delivered` and `order.completed` post-commit webhook behavior and failed-transition guards.

The contract tests use the existing mock-oriented producer patterns rather than introducing another integration framework. Backend remains the authority; client realtime handlers remain convergence mechanisms.

### Marketplace reachability audit

The marketplace Flutter audit classified the concrete marketplace screen graph as:

- **18 CONFIRMED REACHABLE** through explicit production routing;
- **7 INDIRECTLY REACHABLE** through active parent/import construction paths;
- **2 DEAD**, after constructor, router, dynamic-navigation, string-token, notification, and deep-link searches.

The two proven-dead screens were removed in a separate Frontend PR #38. No marketplace Prisma model was deleted because no model met the proof threshold for safe removal.

The audit and deletion were deliberately separated so reachability evidence remains reviewable independently from destructive cleanup.

### Five-repository duplication audit

The four application repositories were audited for duplicate API wrappers, realtime listeners, event constants, financial movement implementations, compatibility layers, and contradictory state transitions.

No duplicate was found with sufficient evidence to justify immediate consolidation. Several superficially repeated structures are intentional boundaries: Backend authority versus projections, Admin financial facade versus generic transport, centralized realtime ownership, payment failover providers, and compatibility naming.

### Issue hygiene

- Business Portal CI issue #4 was closed after verifying active CI execution.
- Admin Portal typecheck issue #8 was closed after verifying the typecheck gate runs in CI and passes.

### Request correlation

Backend PR #82 adds the minimum correlation foundation without introducing a second tracing system:

- existing request-ID middleware remains the canonical HTTP boundary;
- `AsyncLocalStorage` carries the active request ID to downstream code;
- invalid/unbounded inbound IDs are rejected in favor of a fresh UUID;
- `X-Request-Id` response behavior is preserved;
- Pino structured logs receive the active request ID through `mixin()`;
- financial realtime events are decorated through the existing Socket.IO singleton rather than a second transport;
- focused tests cover request propagation, module isolation, logging context, and financial/non-financial event boundaries.

The design intentionally avoids threading Express request objects through financial domain APIs.

## Verification rule

A hardening item is considered **VERIFIED** only when the exact implementation head has passed the relevant tests and the merge result has passed its own production CI gate. A retry or aggregate PR badge is not sufficient evidence by itself.

## Remaining gate

Verify the post-merge Backend `main` run for merge commit `8323b0cfec1f41cf68c31165d6d38ac6fadf25f2`. It must pass the PostgreSQL-backed Jest suite and database backup/restore drill.

If that gate passes, the pre-UI hardening phase can be marked **VERIFIED** and the next planning cycle should pivot to UI work using the marketplace reachability graph to prioritize live screens.

## Guardrails retained

- no new Prisma models;
- no new product verticals during hardening;
- no ad-hoc code-rewrite workflows;
- no client-side financial authority;
- financial mutations remain atomic and backend-authoritative;
- realtime remains a convergence signal;
- contract changes require producer and consumer tracing;
- destructive dead-code removal requires proof and a separate PR;
- expensive CI gates remain PostgreSQL-backed and must be independently verified after merge.
