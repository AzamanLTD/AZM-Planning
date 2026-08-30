# Realtime Convergence Deep Dive — 2026-08-30

**Status:** VERIFIED / CONTINUING
**Scope:** Backend → Admin Portal → Business Portal → Flutter

## Purpose

Record the research and implementation decisions from the current cross-repository realtime hardening cycle so future work does not duplicate existing infrastructure or lose important race/convergence findings.

## Verified architecture

The backend remains authoritative for domain and financial state. Socket.IO delivery is a convergence signal. Clients should invalidate/refetch canonical API queries instead of treating socket payloads as a second financial source of truth.

The current transports are intentionally shared:

- Backend: existing authenticated Socket.IO server and domain notification/event producers.
- Business Portal: existing singleton socket + `realtimeQueryBridge.js`.
- Admin Portal: existing authenticated admin socket + `useAdminRealtime.js`.
- Flutter: existing singleton `SocketService` + existing realtime deduper/callback registry.

No second event bus, socket transport, financial state store, or parallel cache was introduced during this cycle.

## Completed implementation evidence

### Backend

- Tracking authorization/telemetry/timestamp hardening was merged and verified.
- Business no-show penalty outcome was corrected so `penaltyApplied` reflects the actual financial result, with focused regression coverage.
- Business notification realtime emission is attached to the existing notification persistence chokepoint and is best-effort after persistence.
- Existing SmartEscrow database terminal-state protection was researched before considering another settlement-race fix; no duplicate trigger/state invariant was added.
- Escrow expiry refund convergence was hardened so `escrow_refunded` is emitted only after the atomic refund commits, while the payer's balance signal now reuses the existing canonical `emitBalanceUpdate` helper rather than a second balance payload contract.
- Auto-payout research found a more serious transaction-identity defect: the payout batch worker created a second `TransactionHistory` row with a different provider reference even though the original finance withdrawal already created the canonical pending row. This could make reconciliation ambiguous and could reverse the wrong row on provider failure.

### Business Portal

- Realtime query bridge was made safe across authentication socket replacement.
- Canonical tracking event names (`order:location`, `order:status`, `order:eta`) are consumed while legacy aliases remain for rolling deployment compatibility.
- `invoice_paid` now converges invoice queries rather than incorrectly routing through order invalidation.
- Persisted `ORDER_*` business notifications converge both notification and affected-order queries.
- `escrow_refunded` is now a dedicated order-projection convergence signal. Its escrow/ticket identity is deliberately not interpreted as an order ID; the canonical order roots are invalidated instead.

### Admin Portal

- Existing privileged realtime lifecycle was hardened.
- Financial, escrow and order events converge existing admin query surfaces; socket payloads are not used as authoritative financial values.
- Withdrawal reversal and payout-manual-review admin alerts now use the existing financial invalidation boundary, covering `WITHDRAWAL_AUTO_REVERSED`, `WITHDRAWAL_REVERSAL_FAILED`, and `PAYOUTS_NEED_MANUAL_REVIEW`.

### Flutter

- Tracking listener lifecycle was made identity-safe around the singleton socket.
- Canonical tracking event names were added while legacy aliases remain during rollout.
- Persisted business-notification deduplication remains session/bounded-cache based.
- Business notification feed stale-read handling was further hardened in PR #24: stale initial loads no longer mutate loading state, and cursor-page overlap is de-duplicated by notification ID.
- PR #24 was verified by the complete Flutter Quality run: dependency setup, analysis, and the full test suite completed successfully (273 tests passed).
- PR #24 was squash-merged to `main` as `458a0e9b3edbe5b4a77dc11a4f6fce1ac3f80042`.
- The financial convergence batch now refreshes canonical balance state after financial socket signals rather than accepting socket balance payloads as client truth.

## Important research findings

1. The apparent escrow satisfaction overwrite race is protected at the database layer by an existing SmartEscrow terminal-state transition guard. Do not create a second database invariant without evidence that the existing guard is insufficient.
2. A request-level settlement race can still be a UX/API convergence concern even when database state remains correct. Treat that as response semantics, not as a new financial state machine.
3. Business Portal invoice payment originally used the wrong query-convergence domain. The correct contract is invoice event → invoice query invalidation → canonical invoice API refetch.
4. Flutter business-notification pagination can encounter duplicate IDs at cursor boundaries. Client de-duplication is a presentation/convergence guard; it does not replace server cursor correctness.
5. The Flutter repository currently reports many pre-existing analyzer warnings, but its quality workflow is non-fatal for infos/warnings and the complete test suite is green. Do not mix broad warning cleanup into financial/realtime fixes without a focused batch.
6. `escrowService._refundEscrow` already synchronizes linked BusinessOrder state and creates the durable `ORDER_REFUNDED` business notification. A separate direct business-order mutation must not be introduced merely to consume `escrow_refunded`.
7. Auto-payout must preserve the original `TransactionHistory.txHash` because that value is the canonical provider/idempotency reference created by `financeService.processFiatWithdrawal`. Creating a second transaction row is unsafe and conflicts with the reconciliation worker's guarded identity resolution.

## Current PR state

The current cross-repository batch has these open implementation/history PRs:

- Backend #44 — escrow expiry realtime convergence.
- Backend #45 — canonical transaction identity for auto-payout.
- Business Portal #12 — escrow refund realtime convergence.
- Admin Portal #16 — withdrawal reversal/admin convergence.
- Admin Portal #15 — escrow refund admin convergence.
- Flutter #26 — canonical financial balance convergence.
- Planning #9 — financial convergence batch findings/history.

All are being treated as unmerged until their exact head checks complete. No implementation PR is represented as complete merely because a previous commit or an earlier workflow run was green.

## Current financial event matrix boundary

`authoritative mutation → committed state → canonical event → authorized room → client convergence → canonical refetch → duplicate/reconnect/order handling`

Current verified priorities:

1. escrow settlement/refund/dispute;
2. invoice/payment settlement;
3. withdrawal/provider callbacks;
4. balance mutations;
5. notification delivery and duplication;
6. Admin operational/reconciliation visibility.

## Operating rule for the next loop

Before each implementation, research every producer, transport, consumer, query key, authorization boundary, data identity and existing regression test. Implement only the missing invariant or convergence behavior. After implementation, inspect the complete diff for duplication, run the repository's meaningful CI gate, verify the exact PR head, then perform another cross-repository audit before opening or extending the next batch.
