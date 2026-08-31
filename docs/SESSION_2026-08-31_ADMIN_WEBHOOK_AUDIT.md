# Session 2026-08-31 — Admin Atomicity & Order Webhook Audit

## Purpose

Record the verified current-main state after the escrow convergence merge so future sessions do not repeat or reinterpret this research.

## Escrow convergence

Backend PR #52 (`6ff3016...` head) passed the normal Backend `Azaman Test Suite` workflow (run #270) and was squash-merged to Backend main as `332ff3420d35e927aa7897b63b37352b506e6f26`.

The merged change preserves the SmartEscrow database terminal-state guard and makes concurrent/retry callers converge on an already committed `SETTLED` state without introducing a second settlement path or duplicate settlement socket/system-message emission.

Post-merge review must still compare the resulting mainline against all existing escrow producers/consumers before further escrow work.

## Admin force-release audit

Verified current Backend main path:

`controllers/adminController.js::forceRelease` reads a `DISPUTED` trade, performs a separate conditional `DISPUTED -> PAID` write, then calls `services/p2p.service.completeTrade`.

`completeTrade` itself atomically claims `PAID -> COMPLETED` inside the financial transaction before moving escrow/balances and recording `SystemProfitFees`, `AdminProfitLog`, and `TransactionHistory`.

The separate Admin `DISPUTED -> PAID` write is therefore the remaining transaction-boundary defect. A failed `completeTrade` can strand the trade in `PAID`; a blind rollback can overwrite a newer legitimate state.

A guarded automation experiment was attempted through a temporary PR #53, but GitHub did not execute the branch-local workflow. The PR was closed without production changes, and its branch was reset to the merged Backend main commit. No experimental source patch reached main.

Do not treat PR #53 as implementation work. The correct implementation remains to make Admin force release claim `DISPUTED -> COMPLETED` inside the same canonical financial transaction used by `completeTrade`, or otherwise achieve the same atomicity without duplicating settlement logic. Any implementation must preserve normal PAID completion behavior for non-admin callers.

Required regression coverage:
- two concurrent Admin force-release calls produce one settlement and one loser response;
- final trade is COMPLETED;
- exactly one escrow decrement/receiver credit/fee/history set is produced;
- synthetic settlement failure leaves the trade DISPUTED, never PAID;
- no duplicate post-commit notifications/messages are produced.

## Order webhook audit

Verified current Backend main:

- `businessOrderService.markDelivered` atomically transitions `PAID -> DELIVERED`.
- `businessOrderController.markDelivered` then calls `webhookController.triggerEvent(..., 'order.completed', ...)` while the persisted order is still DELIVERED.
- `businessOrderService.updateOrderStatusFromEscrow` maps `SETTLED`/`RELEASED` to `COMPLETED`, including `DELIVERED -> COMPLETED`.
- `businessOrderService` already imports `emitWebhookEvent` and uses it for `order.created`, but the escrow-driven completion path does not emit `order.completed`.
- `webhookEmitter.js` documents `order.delivered` but not `order.completed`.
- `webhookController.listEvents` advertises `order.completed` but not `order.delivered`.
- `webhookController.triggerEvent` dispatches the exact event string selected by each webhook endpoint; adding both aliases would cause duplicate external deliveries rather than resolve the semantic mismatch.
- Business Portal's singleton socket/realtime bridge handles internal realtime invalidation (`business_order_delivered`, order status/location/ETA, escrow events) and canonical HTTP query invalidation. It is not an external webhook consumer for `order.completed`/`order.delivered`.
- Business Portal Orders UI already treats DELIVERED and COMPLETED as distinct states.

Target contract to implement after the remaining producer/consumer audit:

`PAID -> DELIVERED` → external `order.delivered` exactly once.

`DELIVERED -> COMPLETED` → external `order.completed` exactly once, after the authoritative completion mutation commits.

`order.created` remains tied to creation.

No compatibility alias should be added unless an actual existing consumer is identified.

## Continuation rule

Before changing either boundary, re-read all affected Backend producers/tests and all Admin, Business Portal, and Flutter consumers. Then implement one canonical path, add failure/concurrency regression coverage, run exact-head CI, merge only when green, and re-audit current main for duplicate events/listeners/transaction paths.
