# AZAMAN — Current Engineering State — 2026-08-31

## Purpose

Durable continuation point for the AZM engineering program. This state is verified against the live GitHub repositories after the previous session reached its chat limit.

## System objective

AZM is a multi-surface financial platform:

`AZM-backend ↔ AZM-adminPortal ↔ AZM-businessPortal ↔ AZM-frontend (Flutter)`

`AZM-Planning` is the persistent architecture/verification record.

Engineering standard: correctness, atomic financial operations, authoritative contracts, realtime convergence, idempotency, authorization, observability, testability, scalability, clean architecture, minimal duplication, backwards compatibility, and safe incremental delivery.

## Continuation protocol

For every meaningful change:

`research → implement → check → research → implement → test → audit`

Before editing, trace all affected producers/consumers across the complete system. Never invent an API contract. Never duplicate an existing wrapper/service/listener/helper. CI is a gate, not decoration. Financial mutations require explicit review of transaction boundaries, race conditions, balances, fees, audit history, notifications, realtime timing, rollback and HTTP error semantics.

## Verified repository / branch baseline

Live branch search on 2026-08-31:

- `AZM-backend`: `main` only.
- `AZM-frontend`: `main` only.
- `AZM-Planning`: `main` only.
- `AZM-adminPortal`: `main` plus `refactor/financial-api-escrow-consumer` (merged PR branch; no longer active work).
- `AZM-businessPortal`: `main` plus `test/business-portal-smoke-foundation` (merged PR branch; no longer active work).

The two remaining non-main branches are historical merged branches. The available GitHub connector exposes branch creation/ref movement but not branch-ref deletion; do not treat these as active engineering work or resurrect them.

## Verified open PR baseline

Current live search shows no open PRs after the two continuation PRs below were merged.

## Completed in this continuation

### Admin Portal — Smart Escrow financial boundary

PR #20: `refactor: route escrow disputes through financial API facade`

Merged commit: `bdabd790acf4f1d3239429f4f703360083cb0f8a`.

Before implementation, the Backend producer was audited through:

`routes/adminRoutes.js → controllers/adminController.js → escrowService`

The Backend `GET /api/admin/escrow-disputes` producer returns `success`, `disputes`, and pagination `{ page, limit, total, totalPages }`; each dispute includes the escrow, ticket, payer/payee, raisedBy/assignedTo and ruling fields used by the Admin page.

The Backend resolve path validates the ruling, rejects already-finalized/non-resolvable escrows with HTTP 409, delegates the financial mutation to `escrowService.resolveDispute`, emits `escrow_resolved` after the financial service returns, persists a SYSTEM TicketMessage audit record, closes the parent ticket when appropriate, and writes the append-only admin audit record.

Admin changes merged:

- `EscrowDisputes.jsx` now uses `financialApi.escrow.disputes()` and `financialApi.escrow.resolve(...)` rather than direct `api.js` escrow calls.
- `financialContracts.js` now contains a producer-backed `escrowDisputeListResponseSchema` for the actual Backend response shape.
- `financialApi.js` parses the escrow dispute list through that response contract.
- Existing 30-second refetch behavior, extreme-ruling confirmation, mutation loading state and UI behavior were preserved.
- CI run #52 passed: dependency install, changed-file lint and build all succeeded.

An intermediate agent-generated branch version had accidentally removed the `Issuing…` loading state and changed the DollarSign icon size while also reformatting the file. That branch was discarded/reset to current `main`; the final PR was rebuilt from current `main` and the merged change is a clean 3-file diff with only the intended facade/contract work.

### Business Portal — smoke-test foundation

PR #16: `test: add Business Portal smoke foundation`

Merged commit: `a2977478838a07d03cee900d928846a15e573c7b`.

The smoke foundation:

- adds `scripts/smoke-test.mjs`;
- checks required entry files exist;
- checks critical routes exist in `App.jsx`;
- checks `main.jsx` mounts `<App />`;
- checks `AuthProvider` and `QueryClientProvider` are mounted;
- runs in CI before the existing build.

CI run #32 passed: install, smoke test and build all succeeded.

Dependency hygiene was explicitly corrected before merge. The final diff preserves the original tight ranges:

- `react-resizable: ^4.0.2`
- `glob: ^13.0.6`

No dependency widening is present in the final branch diff. The only package.json addition is the `test:smoke` script.

This smoke test is intentionally a structural foundation, not a substitute for runtime/component/API testing.

## Prior major completed work

### Backend realtime refund producer

Backend PR #55 merged. `_refundEscrow()` now emits `escrow_refunded` after the atomic refund claim commits. Admin and Business consumers were verified. Flutter was deliberately not given a listener because the event was not required on that surface.

Lesson: trace producer → transport → consumers; never assume an event exists merely because a client listens for it.

### Backend Admin force-release atomicity

Backend PR #57 merged and issue #48 closed.

Canonical flow:

- normal: `PAID → COMPLETED`;
- Admin disputed override: `DISPUTED → COMPLETED`.

`p2p.completeTrade()` owns the authoritative atomic claim with `adminOverride`. The Admin controller no longer performs a separate `DISPUTED → PAID` claim. `forceCancel` remains `DISPUTED → CANCELLED` and should not be changed without new evidence.

Required financial properties remain: atomic claim before money movement, authorization at the Admin boundary, concurrency protection, audit/history/profit preservation, correct 409 semantics, and post-commit realtime behavior.

### Historical checkout branch audit

The old retail checkout branch was audited against current Flutter main and rejected as a merge because most of its work already existed on main. Only genuinely unique cart work was identified; the stale branch was removed previously. Do not resurrect it.

## Admin financial API boundary

Current architecture:

`Admin financial consumer`
`↓`
`financialApi`
`↓`
`validated request/response contracts`
`↓`
`src/lib/api.js`
`↓`
`Backend controller/service`

PR #18 established the financial input contract foundation, including identifier validation and escrow resolve validation.

PR #19 migrated the War Room consumer and added the first Backend-backed dispute response contract.

PR #20 extended the boundary to Smart Escrow dispute listing/resolution with an audited response contract.

Do not enable full `src/lib` typechecking prematurely. Continue:

`financial boundary → proven consumers → response contracts → wider type checking`

## Highest-value next Admin financial slices

Research each against Backend before implementation:

1. force release / force cancel consumer paths — confirm current facade coverage and avoid duplicating PR #57's backend mutation logic;
2. withdrawals;
3. payouts;
4. user credit;
5. fee profiles;
6. financial settings.

For each slice: search every Admin consumer call; trace Backend route → controller → service; verify authorization and status/error semantics; verify idempotency/concurrency; verify balance/fee/history/audit/realtime effects where financial; reuse or extend existing contracts; migrate one coherent consumer; test and run exact CI; audit the final diff against `main` and merged PR history.

## Business Portal testing roadmap

The smoke foundation is now merged and is the minimum CI structural gate.

Next, research existing versions/dependencies and choose the smallest compatible test framework rather than blindly installing one.

Preferred progression:

1. build smoke — existing;
2. module/route smoke — now established;
3. auth-state rendering;
4. key page rendering;
5. API/query tests;
6. realtime convergence tests.

Keep the existing React Query + singleton realtime architecture. Do not introduce a second state store or socket layer.

## Cross-repo contract testing

Formal cross-repo contract testing remains a major opportunity.

Priority realtime events include `escrow_refunded`, `escrow_resolved`, business notifications, order events and dispute events actually present in Backend.

For each event record producer location, exact event name, payload schema, emitting condition, transaction timing, Admin/Business/Flutter consumers, deduplication/replay behavior and reconnect behavior.

Priority financial/status contracts include escrow, trade lifecycle, dispute lifecycle, force release, force cancel, withdrawals, payouts, settlement, and error codes/HTTP statuses.

Tests must enforce actual producer contracts rather than convention.

## Marketplace dead-code audit

Still outstanding. Do not delete anything merely because it looks unused. Research references, imports, routes, feature flags, Backend endpoints, Flutter consumers, reachability, duplicated implementations and superseded compatibility layers. Only delete after proving code is unused or superseded.

## Final audit requirements

Before declaring this program phase complete:

- search every repository's branches and remove merged feature branches when repository tooling permits;
- search open/closed/merged PRs for duplicate, superseded and abandoned work;
- search for duplicate API wrappers, realtime listeners, event constants, competing financial implementations, dead code, stale compatibility layers and contradictory status transitions;
- verify `Backend producer → Admin → Business → Flutter` for relevant events and financial transitions;
- inspect the exact latest workflow run for the exact PR head after every meaningful change;
- for every financial mutation verify atomicity, authorization, idempotency, balance mutations, fee calculations, profit logs, transaction history, notifications, socket events, rollback, concurrent requests and HTTP error semantics.

## Immediate continuation

The two continuation slices above are merged and CI-verified. Continue with the next high-risk Admin financial consumer, beginning with a full usage/producer audit before editing. Then progress through the Business Portal runtime test foundation, cross-repo contracts, marketplace dead-code audit, and final whole-system branch/PR/duplication audit.

Do not start over. Do not resurrect stale branches. Do not optimize for merely green CI. The standard is production-grade, coherent, scalable, and boringly reliable.

## Verified live-state addendum — 2026-09-03

The state above predates the September 2026 engineering loop. The following is verified against the live repositories and supersedes the stale open-PR statements above.

### Merged production work

- Backend PR #99 — transit Business OS business scoping. Merge commit `ee14a4a5ae46d9966ef4f92f22f1fe954f0a9044`; exact-head Azaman Test Suite #431 passed, including schema, tests, and DB backup/restore.
- Backend PR #101 — restaurant KDS business scoping. Merge commit `6f0fa2b926555415153ff21ff684f87f02d2fb32`; exact-head Azaman Test Suite #430 passed.
- Backend PR #104 — KDS creation integrity. Merge commit `e9b2a9002c3265ec2d3354d1d8a4da3a99dace3b`; validates location, business-order, and product ownership/activity and adds focused regression coverage.
- Backend PR #106 — dine-in guest-search privacy boundary. Merge commit `0b0c23be39176d8c53ea9cc482e08db36170284b`; exact-head Azaman Test Suite #450 passed.
- Backend PR #105 — hotel Business OS hardening. Merge commit `007bc774a156ba2f9f04d185f02a9d09c8bc2c25`; exact-head Azaman Test Suite #451 passed all 822 tests plus DB backup/restore. The change business-scopes hotel mutations, fixes undefined legacy route service references, requires a registered customer identity for walk-ins, and makes room moves transactional.
- Business Portal PR #38 — hotel front-desk walk-in identity contract. Merge commit `28a22a37f837d8ea83912820b6bf72341974e0d6`; CI #106 passed smoke, tests, and build. The portal now submits public `customerAzamanId` rather than an internal database ID or free-form guest name.
- Frontend PR #74 — authoritative hotel room inventory customer flow. Merge commit `1b9be8b292ff2b59b544f01cd54532c564fbf97b`; Flutter Quality #307 passed.
- Frontend PR #75 — server-authoritative transit booking fare in success UI. Merge commit `9dbe38b0732cb8d45afb6c5a55c409904de3a8be`; Flutter Quality #309 passed after restoring the complete source file from an accidentally truncated automation edit.

### Closed/superseded work

- Backend PR #100, the superseded restaurant KDS business-scope implementation, was closed in favor of the clean PR #101.
- Backend PR #98, the noisy hotel full-file rewrite, remains closed; its semantic findings were reimplemented surgically in PR #105.
- Backend PR #107, an attempted shift-operations hardening patch, was closed without merge because the repository's temporary workflow execution path was unavailable; the isolated branch was reset to the production head. Do not treat it as deployed work.

### New in-flight work

Backend PR #108: `fix(business-os): make employee feedback rating aggregation atomic`.

Current intent:

- keep the existing feedback API contract;
- create feedback and recompute rating inside one Prisma transaction;
- scope the rating aggregate to `businessProfileId` so another business cannot influence the result;
- use a business-scoped employee update for the rating write;
- add focused regression coverage for cross-business rejection and aggregation scope.

At the time of this update, PR #108 is open and its native Azaman Test Suite run is in progress. It must not be merged until the exact current head is green through the DB backup/restore gate.

### Newly verified remaining high-risk area

The HR/shift service still contains raw-ID mutation methods (`updateShift`, `deleteShift`, `clockIn`, `clockOut`, `markNoShow`, and shift-swap claim/approval/rejection) whose service signatures do not consistently require a business scope. Route consumers should be hardened as one coherent slice rather than with isolated checks. This is a known follow-up, not a claimed fix.

The employee service also contains several raw employee-ID methods that should be audited for business scope before being exposed to business operators. In particular, employee stats, EWA operations, and employee CRUD mutations need producer/consumer tracing and concurrency review.

### Updated continuation order

1. Finish PR #108 with exact-head CI and merge only after full verification.
2. Rework the HR/shift mutation hardening as a clean, directly editable patch path, including route authorization and transactional swap approval; do not use unverified temporary workflow commits.
3. Audit employee/EWA mutation concurrency and business scope, prioritizing the financial `requestEWA` balance/ledger sequence.
4. Continue the Business Portal runtime/API testing roadmap against actual production consumers.
5. Complete the cross-repo event/status contract matrix and marketplace dead-code audit.
6. Perform final branch/PR/duplication cleanup using only tooling that can prove the state.
