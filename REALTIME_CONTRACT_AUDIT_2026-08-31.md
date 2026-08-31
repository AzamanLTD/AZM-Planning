# Realtime Contract Audit — 2026-08-31

## Correction: `escrow_refunded`

The previous engineering continuation incorrectly described `escrow_refunded` as a Backend event that already existed in `_refundEscrow()`.

Verified current Backend `main` before implementation:

- `services/escrowService.js::_refundEscrow()` performed the atomic refund claim and balance/history mutation but did **not** emit `escrow_refunded`.
- The historical expiry-worker implementation emitted `escrow_refunded` only from `EscrowExpiryWorker`, meaning the signal existed only for expiry-worker refunds.
- Business Portal `main` already listens for `escrow_refunded` and invalidates the canonical order projection without treating `escrowId` as an `orderId`.
- Admin Portal `main` already includes `escrow_refunded` in its existing escrow event registry.
- Flutter `main` did not listen for the event; that was correct before a Backend producer existed.

## Correct implementation sequence

1. Backend `_refundEscrow()` becomes the single canonical producer.
2. The event is emitted only after the atomic financial transaction resolves successfully.
3. Existing Socket.IO transport is injected once during socket bootstrap; no second transport is introduced.
4. Expiry worker must not emit a second `escrow_refunded` event.
5. Business Portal consumes the event through its existing realtime query bridge.
6. Admin Portal consumes the event through its existing Admin realtime hook.
7. Flutter consumes the event through its existing generic escrow-event dispatch path.
8. Clients treat the event as a convergence signal and refetch authoritative API state; no client becomes a second financial ledger.

## Backend PR

PR #55: `fix(escrow): emit refund convergence from canonical service`

The intended event targets are:

- `user_<payerId>`
- `user_<payeeId>`
- `admin_spy_room`

Payload identity fields:

- `escrowId`
- `ticketId`
- `status`
- `amountUsdc`
- `payerId`
- `payeeId`
- `reason` (`EXPIRY` or `REFUND`)

The transaction must commit before any of these emits. A socket transport error must never turn a committed financial refund into a failed financial operation.

## Flutter PR

PR #31: `fix(realtime): receive canonical escrow refund event`

Only the existing escrow event registry changes. No new socket connection or local financial state is introduced.

## Business/Admin verification

Business Portal `main` already contains `escrow_refunded` in the singleton realtime bridge and has a dedicated regression test from the previously merged convergence work.

Admin Portal `main` already contains `escrow_refunded` in `useAdminRealtime.js` and routes it through the existing escrow invalidation path.

## Branch protection

Verified on 2026-08-31 that `main` is currently unprotected in:

- AZM-backend
- AZM-businessPortal
- AZM-frontend
- AZM-adminPortal

The available GitHub connector can read protection state but does not expose the required branch-protection write endpoint. Therefore protection cannot be safely configured from this session without inventing an unsafe workaround or asking for credentials.

Required policy once repository administration write access is available:

- protect `main`;
- require pull requests;
- require the repository's canonical CI check(s) to pass;
- require branches to be up to date before merge where appropriate;
- block force-push and branch deletion;
- dismiss stale approvals on new commits;
- require conversation resolution;
- keep administrators subject to the protection where the organization policy permits.

Do not modify CI merely to satisfy protection. First identify the actual successful check names for each repository, then configure those exact checks.

## Branch inventory

The four application repositories currently contain approximately 144 branches between them, with additional Planning branches bringing the organization total to the previously observed 153.

Many are clearly historical feature/fix branches. The current GitHub connector exposes branch creation/ref movement but not branch deletion, so stale branches cannot be safely deleted through the available action. Do not reset potentially meaningful historical branches merely to make the count smaller.

Temporary branches created during the current engineering session must be treated as disposable and removed/auto-deleted once their PRs are merged or closed:

- Backend `fix/escrow-refund-service-emission-20260831`
- Backend `test/escrow-refund-service-realtime` (superseded experiment)
- Business Portal `ci/deterministic-npm-install-20260831`
- Flutter `fix/realtime-escrow-refunded-convergence-20260831`

## Definition of done for this contract

- Backend PR passes existing CI.
- Flutter PR passes existing CI.
- Business Portal existing CI hardening remains green.
- Admin/Business/Flutter all consume the same event name and identity contract.
- No duplicate producer remains in the expiry worker.
- Final PR diffs are audited against merged and closed PRs.
- After merge, reread each affected `main` implementation and update this document with the final SHAs.
