# Financial Convergence Batch 2 — 2026-08-30

## Scope

Research was performed across the Backend, Business Portal, Admin Portal, Flutter app, and planning repository before implementation. The goal was to preserve one authoritative backend ledger while making every connected client converge immediately after committed financial transitions.

## Withdrawal lifecycle finding

The withdrawal schema contains a durable `network` field and an additive `transactionHistoryId` identity bridge. The manual fiat withdrawal path already creates the canonical `TransactionHistory` row and uses its `txHash` as the provider correlation reference.

The auto-payout worker was nevertheless creating a second transaction-history row with a newly generated reference. That broke the canonical identity invariant and could make reconciliation ambiguous. The implementation removes that duplicate creation, reuses the existing transaction reference, and refuses ambiguous/missing correlation.

A second lifecycle gap was found during research: auto-payout marked the Withdrawal `PROCESSING` only after provider I/O. That permitted concurrent worker instances to dispatch the same PENDING withdrawal. The worker now atomically claims PENDING → PROCESSING before provider I/O.

A third gap followed from that fix: reconciliation previously scanned only PENDING withdrawals. It now scans both PENDING and PROCESSING, so a process crash after the claim does not strand a user's reserved funds.

The auto-payout provider call also now carries the stored withdrawal network so the existing Moolre routing contract (MTN, Telecel/Vodafone, AirtelTigo) is preserved rather than defaulting an automatic payout to MTN.

## Invoice lifecycle finding

Business invoices already had a deterministic `payTxHash` (`INV_PAY_<invoiceId>`) and a Business Portal realtime query bridge. Research showed that the service checked the idempotency marker before entering the database transaction. Two concurrent payment requests could therefore both observe `payTxHash = null` before either request committed the marker and both could perform financial mutations.

The settlement transaction now atomically claims the invoice while it is still SENT and unclaimed. The claim is rolled back with the financial transaction if a later step fails. A concurrent loser re-reads the committed PAID invoice and returns an idempotent replay without another debit/credit.

The existing `invoice_paid` event is emitted post-commit to both the payer and business owner rooms. Business Portal already has the correct dedicated invoice invalidation path, so no duplicate business-side realtime mechanism was added.

Flutter research found that `invoice_paid` is currently routed through the generic escrow callback but has no active invoice-provider consumer. This remains a client convergence follow-up: the existing singleton socket boundary should be extended without creating a second socket or local financial source of truth.

## Cross-repository verification notes

- Backend remains the financial source of truth.
- Socket events are convergence signals, not authoritative balances or invoice state.
- Business Portal's singleton realtime/query bridge already separates order, escrow, invoice, and notification invalidation.
- Admin Portal's realtime boundary already covers financial settlement and reconciliation exceptions.
- Flutter uses one singleton SocketService; financial balance events already trigger canonical REST refreshes.
- No new financial ledger, duplicate refund guard, second socket connection, or client-side financial state machine was introduced.

## Open implementation boundaries

- Backend #45: withdrawal canonical identity / atomic auto-payout claim / PROCESSING reconciliation.
- Backend #46: atomic invoice payment idempotency and bilateral invoice-paid convergence.
- Flutter follow-up: consume `invoice_paid` through the existing singleton callback registry and refresh `myInvoicesProvider` from the canonical API.

## Verification policy

No PR is treated as complete merely because the diff is logically correct. Exact-head repository workflows must pass before merge. After merge, the resulting main branch is re-researched for duplicated logic and downstream consumers before the next implementation boundary is selected.
