# AZAMAN — 2026-08-31 Continuation Progress: Financial Boundaries

## Status
VERIFIED MERGED BATCHES + NEXT WORK IN PROGRESS

This record updates the persistent engineering history after the chat-session continuation. It is additive; historical planning documents remain intact.

## Plan alignment

The active master sequence remains:

1. Admin financial consumer boundary hardening;
2. Business Portal runtime test foundation;
3. cross-repository financial/event contract verification;
4. marketplace dead-code/supersession audit;
5. final whole-system branch/PR/duplication audit.

This sequence remains subordinate to the master P0 financial truth/reconciliation workstream. No unrelated feature vertical work was intentionally started.

## Completed in this continuation

### Admin Portal — War Room financial boundary

PR #84: `refactor(financial): route War Room trade mutations through facade`

Merged commit: `7367ebb77b2a74de5fbcab3e3179703358a31935`.

Research established that War Room still bypassed the existing `financialApi` facade for force release, force cancel, normal dispute resolution, and admin message injection. The live-trades read was deliberately left on the legacy transport because no audited response contract existed for that endpoint.

The final migration:

- routes force release through `financialApi.disputes.forceRelease`;
- routes force cancel through `financialApi.disputes.forceCancel`;
- routes dispute resolution through `financialApi.disputes.resolve`;
- routes admin message injection through `financialApi.disputes.injectMessage`;
- preserves the existing `trades.live()` transport until its response contract is audited.

### Admin/Backend contract correction — admin notes

Research uncovered a real cross-repository request mismatch:

- Admin facade exposes an operator-friendly `reason` argument;
- Backend `forceRelease` / `forceCancel` controllers and the existing validation schema use `adminNotes`.

The Admin transport now maps `reason` → `adminNotes`. No backend financial mutation logic was changed.

This prevents operator notes from being silently discarded and keeps the facade contract separated from legacy transport naming.

### Backend — force-cancel request validation

PR #61: `fix(admin): validate force-cancel request contract`

Merged commit: `9fe12df0ea1615e2508e9ec231fb98b938be51f6`.

The existing `forceReleaseSchema` is now also applied to `/api/admin/disputes/force-cancel`.

The existing atomic cancellation implementation remains unchanged:

`DISPUTED → CANCELLED` is conditionally claimed before escrow refund work, concurrent losers receive HTTP 409, and audit/realtime behavior remains post-commit.

Verification: the exact PR head passed the complete PostgreSQL-backed `Azaman Test Suite`, including database schema setup, transaction-quote overlay installation, Prisma generation and the full serial Jest suite.

### Admin Portal — frozen withdrawal response contract

A producer/consumer audit found that `/api/admin/withdrawals/pending` returns two different collections:

- `pending`: `Withdrawal` records with withdrawal-specific review fields;
- `frozen`: `TransactionHistory` records with a smaller common identity/status shape and transaction-history fields.

The Admin financial facade had incorrectly parsed both arrays with `withdrawalSchema`. That meant the presence of a valid frozen history row could cause the entire response parse to fail.

The corrected contract gives `frozen` its own producer-backed schema while keeping `pending` strict.

PR #85 was closed and rebuilt against the updated `main` after a base movement caused by temporary-workflow cleanup. The final source fix is tracked for re-verification/merge from the current mainline rather than relying on stale PR metadata.

## Verification state

### Admin #84

Exact validation job passed:

- dependency install: success;
- changed-file lint: success;
- typecheck: success;
- production build: success.

### Backend #61

Exact validation job passed all steps, including the complete PostgreSQL-backed test suite.

### Temporary workflow hygiene

A temporary guarded migration workflow accidentally became part of the Admin PR merge commit. This was detected during post-merge repository inspection and removed from `main` in commit `16b8eb9ab09880b157f993f89d02493bdcb8cd6e`.

This is recorded as an engineering-process correction: temporary mutation workflows must never remain in production branches.

## Business Portal — current next batch

A runtime AuthProvider test suite has been implemented on branch `test/auth-runtime-state` and PR #18 is open.

Coverage:

- business session restore → profile load → realtime wiring;
- failed restore fails closed and clears stale persisted session state;
- admin restore + persisted business selection;
- admin login + first-business selection;
- logout + socket disconnect + persisted-session cleanup.

The repository already has Vitest + React Testing Library + jsdom configured, so no second test framework or dependency family was introduced.

Important limitation: Business Portal GitHub Actions currently produces no workflow runs for the branch/repository. Therefore the PR is not marked or treated as CI-verified. Static review and repository-local test design continue, but no false green claim is made.

## Next loop

1. Verify the rebuilt frozen-withdrawal contract against current `main` and add/merge regression coverage.
2. Complete the Business Portal runtime testing progression where CI availability permits.
3. Build the financial event matrix across:
   - escrow funded/satisfied/disputed/settled/refunded;
   - invoice payment;
   - withdrawal progress/settlement/failure;
   - balance mutation;
   - admin reconciliation exceptions;
   - duplicate delivery/reconnect;
   - stale terminal-state ordering.
4. For each event verify:
   `producer → authorization → room → payload identity → consumer → canonical read/invalidation → deduplication → reconnect → terminal ordering → tests`.
5. Continue with the marketplace dead-code audit only after financial/event convergence work is sufficiently covered.

## Intentional non-goals / parked items

- Do not resurrect stale checkout branches.
- Do not enable full Admin `src/lib` typechecking prematurely.
- Do not introduce a second socket/event bus/state store.
- Do not make Flutter consume events it does not actually need.
- Do not reopen GitHub branch-protection work that is already parked due plan/tier constraints.
- Do not claim Business Portal CI verification while GitHub Actions remains unavailable.
