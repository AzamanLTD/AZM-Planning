# Financial Convergence Batch 2 — 2026-08-30/31

## Scope

Research was performed across the Backend, Business Portal, Admin Portal, Flutter app, and planning repository before implementation. The goal was to preserve one authoritative backend ledger while making every connected client converge immediately after committed financial transitions.

## Withdrawal lifecycle finding

The withdrawal schema contains a durable `network` field and an additive `transactionHistoryId` identity bridge. The manual fiat withdrawal path already creates the canonical `TransactionHistory` row and uses its `txHash` as the provider correlation reference.

The auto-payout worker was nevertheless creating a second transaction-history row with a newly generated reference. That broke the canonical identity invariant and could make reconciliation ambiguous. The implementation removes that duplicate creation, reuses the existing transaction reference, and refuses ambiguous/missing correlation.

A second lifecycle gap was found during research: auto-payout marked the Withdrawal `PROCESSING` only after provider I/O. That permitted concurrent worker instances to dispatch the same PENDING withdrawal. The worker now atomically claims PENDING → PROCESSING before provider I/O.

A third gap followed from that fix: reconciliation previously scanned only PENDING withdrawals. It now scans both PENDING and PROCESSING, so a process crash after the claim does not strand a user's reserved funds.

The auto-payout provider call also now carries the stored withdrawal network so the existing Moolre routing contract (MTN, Telecel/Vodafone, AirtelTigo) is preserved rather than defaulting an automatic payout to MTN.

The first exact-head backend CI run exposed one compatibility regression introduced while generalizing the provider label: the reconciliation worker used the neutral `DISBURSEMENT` label while `ProviderSettlementAttempt` intentionally allowed only the existing `MTN_MOMO_DISBURSEMENT` and `MOOLRE` identities. The fix normalizes `DISBURSEMENT` to `MTN_MOMO_DISBURSEMENT` rather than creating a third provider namespace, with a dedicated regression test. Backend PR #47 subsequently passed its exact-head test gate and was merged.

## Invoice lifecycle finding

Business invoices already had a deterministic `payTxHash` (`INV_PAY_<invoiceId>`) and a Business Portal realtime query bridge. Research showed that the service checked the idempotency marker before entering the database transaction. Two concurrent payment requests could therefore both observe `payTxHash = null` before either request committed the marker and both could perform financial mutations.

The settlement transaction now atomically claims the invoice while it is still SENT and unclaimed. The claim is rolled back with the financial transaction if a later step fails. A concurrent loser re-reads the committed PAID invoice and returns an idempotent replay without another debit/credit.

A second invoice defect was found during final PR review: the original status guard ran before the `payTxHash` replay check, meaning a legitimate retry of an already-PAID invoice could still be rejected. The replay check now runs first, and a dedicated regression test verifies that a committed PAID invoice is returned without entering another financial transaction. Backend PR #46 passed its exact-head test gate and was merged.

The existing `invoice_paid` event is emitted post-commit to both the payer and business owner rooms. Business Portal already has the correct dedicated invoice invalidation path, so no duplicate business-side realtime mechanism was added.

## Flutter invoice and order convergence

Flutter research traced `invoice_paid` through the existing singleton `SocketService`, `MyInvoicesScreen`, and `myInvoicesProvider`. The socket already received the event, but it routed it through the generic escrow callback, leaving the customer invoice provider without a dedicated convergence consumer.

Flutter PR #27 added a dedicated `invoice_paid` callback to the existing singleton, removed the event from the generic escrow path, and safely registered/unregistered the invoice-screen consumer. The consumer refreshes `myInvoicesProvider` from the canonical API instead of trusting socket payloads as invoice state. It passed Flutter Quality and was merged.

A follow-up research pass found that `invoice_received` was already emitted by the backend after a business sends an invoice but had no dedicated Flutter consumer. Flutter PR #28 added the dedicated singleton callback and canonical invoice refresh, passed Flutter Quality, and was merged.

The same research-first pass found that `business_order_delivered` was already emitted by the backend after the authoritative PAID → DELIVERED mutation, while My Orders did not consume it. Flutter PR #29 added the customer-side canonical refresh. It passed Flutter Quality and was merged.

## Cross-repository verification notes

- Backend remains the financial source of truth.
- Socket events are convergence signals, not authoritative balances or invoice/order state.
- Business Portal's singleton realtime/query bridge separates order, escrow, invoice, and notification invalidation.
- Admin Portal's realtime boundary covers financial settlement and reconciliation exceptions.
- Flutter uses one singleton SocketService; financial balance events trigger canonical REST refreshes.
- No new financial ledger, duplicate refund guard, second socket connection, or client-side financial state machine was introduced.
- Superseded/non-mergeable implementation PRs are closed rather than left alongside their current-main replacements.

## Current implementation boundaries

- Withdrawal canonical identity, atomic auto-payout claiming, PROCESSING reconciliation, network propagation, and provider-label compatibility are merged through Backend PR #47.
- Atomic invoice payment idempotency, committed replay behavior, and bilateral invoice-paid convergence are merged through Backend PR #46.
- Dedicated Flutter invoice and order convergence consumers are merged through Flutter PRs #27, #28, and #29.
- The next research boundary is the order/escrow realtime subscription architecture, especially ticket-scoped consumers that currently use direct raw-socket listeners alongside the singleton dispatcher. No implementation should begin until the full affected path is traced and duplication/reconnect behavior is understood.

## Verification policy

No PR is treated as complete merely because the diff is logically correct. Exact-head repository workflows must pass before merge. After merge, the resulting main branch is re-researched for duplicated logic and downstream consumers before the next implementation boundary is selected.
