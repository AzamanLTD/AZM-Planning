# Admin Force-Release Atomicity — Deep Dive

Date: 2026-08-31
Issue: AZM-backend #48
Status: research complete; implementation intentionally not yet committed until the canonical transaction boundary is changed surgically.

## Current production path

Route:
`POST /api/admin/disputes/force-release`

Route stack:
`protect` → `adminOnly` → `validate(forceReleaseSchema)` → `adminController.forceRelease`.

The route is therefore already admin-only and request-shape validated.

## Current controller defect

`controllers/adminController.js::forceRelease` currently:

1. Reads the trade outside a transaction and requires `DISPUTED`.
2. Determines the counterparty that would normally release the trade:
   - SELL → vendor
   - BUY → buyer
3. Performs a separate atomic `DISPUTED → PAID` `updateMany` claim.
4. Calls `p2pService.completeTrade()` in a second transaction.
5. Creates/updates the admin intervention conversation/message outside that financial transaction.
6. Emits balance updates and `trade_update` after the service call.

The existing H9 comment correctly identifies the stranded-`PAID` problem: if the claim commits and the delegated completion subsequently fails, the trade can remain `PAID` without the settlement having completed.

That is the central defect. The status claim and financial settlement are not one atomic unit.

## Current P2P settlement path

`services/p2p.service.js::completeTrade()` currently:

- pre-reads the trade;
- requires `PAID`;
- authorizes the normal releasing counterparty;
- calculates the active fee profile and fee split;
- starts its own Prisma interactive transaction;
- atomically claims `PAID → COMPLETED` before balances;
- decrements the releasing party's escrow;
- credits the recipient's available balance net of admin fee;
- increments SystemProfitFees;
- writes AdminProfitLog;
- writes buyer TransactionHistory;
- commits;
- then performs non-blocking double-entry journal calls;
- returns post-commit notification data for the controller.

This service is already the canonical financial settlement engine and MUST remain the single implementation. A second admin-specific payout algorithm would create financial drift and is prohibited.

## Required target architecture

Admin force-release must become one atomic financial transaction:

`DISPUTED claim → COMPLETED claim → escrow decrement → recipient credit → admin fee → profit log → transaction history`

If any financial operation fails, the trade must remain `DISPUTED` and all balance/ledger mutations must roll back.

The admin intervention message, socket events, push notifications, and balance broadcasts must happen only after the financial transaction commits.

The admin override must be an authorization input to the canonical settlement service, not a second settlement implementation.

## Concurrency contract

Two simultaneous Admin force-release calls on the same DISPUTED trade:

- exactly one may win;
- winner commits the full settlement;
- loser receives HTTP 409;
- no double balance credit;
- no double escrow decrement;
- no double admin fee;
- no duplicate TransactionHistory financial settlement;
- no duplicate AdminProfitLog;
- no duplicate post-commit notifications.

A failed winner must not consume the DISPUTED claim permanently. A retry after a failed transaction must still see DISPUTED and be able to attempt the settlement again.

The existing H8 `PAID → COMPLETED` claim in `completeTrade()` is not sufficient for the admin path because the current controller commits a separate `DISPUTED → PAID` claim first.

## Important architectural constraint discovered during research

Do NOT solve this by wrapping the existing `completeTrade()` in an outer transaction while using a proxy to flatten its nested `$transaction` call unless the transaction boundary and post-commit journal behavior are explicitly redesigned.

The current `completeTrade()` deliberately performs journal writes after its own transaction resolves. Calling it inside an outer transaction would move those side effects into the outer transaction's callback and can cause post-settlement side effects to occur before the final outer commit. Prisma documents that external side effects inside an interactive transaction callback can survive a later rollback; the recommended pattern is to return data from the transaction and perform side effects after commit.

Therefore the clean solution is to make the canonical service itself transaction-aware rather than introducing an ambient/proxy transaction workaround.

## Preferred implementation

Refactor `completeTrade()` so the settlement transaction can accept a controlled authorization/transition context, e.g.:

`completeTrade(prisma, { tradeId, releasedByUserId, adminOverride })`

The service should:

1. Read immutable trade facts needed for calculation/authorization.
2. For normal user release, require PAID and the existing counterparty authorization.
3. For `adminOverride`, require the caller to have already been authenticated/authorized by the Admin route and allow the source status `DISPUTED`.
4. Inside ONE transaction, perform a single conditional claim:
   - normal: `PAID → COMPLETED`;
   - admin: `DISPUTED → COMPLETED`.
5. Run all existing financial mutations against the transaction client.
6. Return notification/journal/event data from the transaction.
7. Only after commit, execute journal writes and notification/event delivery.

The service should not trust a boolean supplied by an arbitrary HTTP caller; only the Admin controller/route should invoke the admin override path. The service API should make the distinction explicit enough that ordinary P2P callers cannot accidentally bypass the normal authorization rule.

## Controller target

`adminController.forceRelease` should no longer perform `DISPUTED → PAID` itself.

It should:

- validate the trade exists and is DISPUTED for early user feedback;
- call canonical `completeTrade(..., adminOverride: true)`;
- create the admin intervention message only after the financial service commits;
- emit balance/trade updates only after commit;
- map `TRADE_ALREADY_FINALIZED`/claim conflicts to 409;
- preserve the existing audit record.

The pre-read is acceptable for UX/error classification, but it must never be relied upon for concurrency correctness.

## Existing historical work / duplication audit

PR #53/#54 were previously used for an attempted guarded implementation. They were closed/reset and do not represent current source. Their final visible commits contained workflow scaffolding rather than a landed implementation.

Do not resurrect their self-modifying workflow. Do not copy a second settlement algorithm from those experiments.

The current canonical implementation is H8's atomic `PAID → COMPLETED` settlement in `p2p.service.js`; the admin fix must extend that canonical path.

## Tests required before merge

Database-backed regression suite:

1. `DISPUTED` admin force-release succeeds and settles exactly once.
2. Two concurrent admin force-release requests produce `[200, 409]` in either order.
3. Concurrent loser does not mutate balances/history/profit.
4. Injected failure during settlement leaves trade `DISPUTED`.
5. Injected failure after the claim but before a balance mutation rolls the claim back.
6. A retry after a failed settlement can succeed.
7. Normal user `completeTrade` behavior remains unchanged.
8. Normal user cannot release a `DISPUTED` trade merely because the admin path now supports it.
9. No duplicate admin intervention message on concurrent calls.
10. No duplicate post-commit socket notification on concurrent calls.
11. Financial TransactionHistory count remains exactly one for the successful settlement.
12. AdminProfitLog count remains exactly one for the successful settlement.
13. SELL and BUY trade directions both covered.

## Cross-surface verification after backend merge

Admin Portal:
- verify 409 causes the UI to refresh/reconcile rather than show a false success;
- verify completed trade appears immediately from the existing realtime path.

Business Portal:
- verify disputed → completed convergence and balance refresh.

Flutter:
- verify the existing trade update/balance event paths converge without adding a second socket transport.

## Engineering decision

Do not implement the old two-step `DISPUTED → PAID → completeTrade()` sequence again.

Do not create an admin-only duplicate settlement engine.

Do not use a self-modifying CI workflow to patch source code.

Do not add client-side compensation for a backend atomicity defect.

The next implementation should modify the canonical service boundary and the Admin controller together, then run the focused DB-backed race/failure tests before touching any other surface.
