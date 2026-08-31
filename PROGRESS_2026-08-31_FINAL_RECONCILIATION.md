# AZAMAN — 2026-08-31 Final Reconciliation

## Current verified state

### Backend
- PR #61 `fix(admin): validate force-cancel request contract` is merged at `025ef4faca24cfcfe52cb49c3a0aed3742ac681b`.
- Exact PR head passed the full PostgreSQL-backed Azaman Test Suite before merge.
- `main` already contains the canonical business-order delivery/completion semantics from PR #59; superseded #60 is not to be resurrected.

### Admin Portal
- PR #84 `refactor(financial): route War Room trade mutations through facade` is merged at `7367ebb77b2a74de5fbcab3e3179703358a31935`.
- PR #84 passed changed-file lint, typecheck and production build.
- The accidentally merged temporary War Room guard workflow was removed from Admin `main` in `16b8eb9ab09880b157f993f89d02493bdcb8cd6e`.
- PR #86 `fix(financial): harden withdrawal response contract` is merged at `62a8f322611991833f72b459d02c6399ef442a72`.
- PR #86 passed the complete Admin lint/typecheck/build gate.
- The frozen-withdrawal response contract now matches the Backend producer: `pending` is parsed as Withdrawal rows; `frozen` is parsed as TransactionHistory-backed rows.
- Regression tests were added for the frozen response and for strict pending-field validation.

### Business Portal
- PR #18 `test: cover Business Portal auth state transitions` is open on `test/auth-runtime-state`.
- The suite covers business session restoration, fail-closed restore, admin business selection, admin login and logout/realtime cleanup.
- Existing Vitest/React Testing Library/jsdom infrastructure is reused; no second test stack was introduced.
- GitHub Actions is currently returning zero workflow runs for this repository/branch. PR #18 therefore remains unverified and must not be treated as green or merged solely from static review.

### Flutter
- Existing event/balance convergence hardening is already merged, including the stale business-notification refresh race fix PR #22.
- The singleton SocketService already exposes `withdrawal_progress` and `withdrawal_settled` and refreshes canonical balance after settlement.

## Research conclusions carried forward

- Do not add new escrow lifecycle producers: current Backend `main` already emits `escrow_funded`, `escrow_pending_settlement`, `escrow_settled`, and `escrow_refunded` after committed mutations, with atomic single-winner state claims.
- Do not create another order webhook implementation: PR #59 is canonical on Backend `main`.
- Do not create another balance/event transport: Flutter already converges financial socket signals through canonical balance reads.
- Do not resurrect stale branches or closed superseded PRs.

## Exact next workstream

1. Resolve or explicitly document Business Portal GitHub Actions availability and continue its runtime-test progression where executable.
2. Complete the cross-repository financial event matrix with explicit producer → auth scope → room → payload identity → consumer → canonical read/invalidation → duplicate/reconnect behavior → terminal-state ordering → tests.
3. Only after financial/event convergence is sufficiently covered, run the marketplace dead-code/supersession audit.
4. Finish with a five-repository current-main audit for duplicate implementations, shadow financial state, stale PRs/branches, workflow contamination, contract drift and planning drift.

## Guardrails

No second socket, event bus, financial shadow store or parallel state machine should be introduced. Backend/database remains authoritative; realtime exists to signal convergence; clients re-read canonical state.
