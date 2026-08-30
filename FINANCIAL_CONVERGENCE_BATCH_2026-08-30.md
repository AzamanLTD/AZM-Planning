# Financial Convergence Batch — 2026-08-30

**Status:** IN PROGRESS / VERIFIED RESEARCH
**Scope:** Backend expiry refund → Flutter financial projection

## Why this batch exists

The realtime architecture requires the backend/database to remain authoritative while Socket.IO events act only as convergence signals. The financial event matrix research found one concrete missing boundary in the SmartEscrow expiry path: the expiry worker commits the refund and closes the ticket, but previously emitted no financial Socket.IO signal afterward.

That meant a connected participant could receive the ticket SYSTEM message yet receive no dedicated terminal escrow/balance convergence signal. The existing financial transaction itself remained atomic; the defect was delivery/convergence after commit.

## Backend finding

`workers/escrowExpiryWorker.js` calls:

`escrowService._refundEscrow(prisma, escrowId, 'EXPIRED')`

The refund service already contains the important atomic terminal-state claim. The worker therefore must not add another financial guard. The missing work is post-commit signalling.

## Backend implementation in PR #44

PR #44 adds post-refund convergence signals only after `_refundEscrow` succeeds:

- `escrow_refunded` → payer user room;
- `escrow_refunded` → payee user room;
- `escrow_refunded` → `admin_spy_room`;
- `balance_update` → payer user room.

The payload contains resource/event identity and terminal status, not a private balance snapshot. Clients are expected to refetch canonical state.

Regression coverage verifies both:

1. successful refund emits the terminal escrow signal and payer balance signal;
2. failed/losing refund emits neither financial convergence signal.

## Flutter implementation in PR #26

The Flutter singleton socket previously treated financial socket payloads as state updates. The batch changes that boundary so:

- `balance_update` triggers canonical balance refresh;
- `deposit_success` triggers canonical balance refresh after its existing callback;
- `withdrawal_settled` triggers canonical balance refresh;
- `azm_reward` and `azm_spend` no longer write socket-supplied balances directly;
- concurrent refresh requests are coalesced;
- balance parsing has focused regression coverage for numeric/string values and legitimate zero values.

No second socket, event bus, financial store, or cache architecture was introduced.

## Important distinction

The backend expiry refund was already financially protected by the existing `_refundEscrow` atomic claim. This batch does **not** add another database trigger or duplicate terminal-state guard.

The new work addresses:

`committed financial mutation → realtime convergence signal → canonical client refetch`

rather than:

`client payload → client financial truth`

## Verification state

- Backend PR #44: open; exact-head CI is running.
- Flutter PR #26: open; exact-head Flutter Quality run is running.
- Earlier Flutter PR #25 was intentionally closed/reset after an intermediate edit introduced unnecessary source rewriting; no code from that discarded version was merged.

## Next research target

Continue the matrix around:

1. admin/manual escrow FULL_REFUND and FULL_RELEASE convergence;
2. withdrawal/provider callback terminal events and balance signalling;
3. invoice payment settlement convergence;
4. Admin reconciliation visibility.

For each target, trace producer → transaction commit → event → recipient authorization → Business Portal → Admin → Flutter → canonical query/state refresh before implementation.
