# AZAMAN — Order & Realtime Hardening Verification — 2026-08-31

## Purpose

Record the final verified state of the order/webhook and cross-repo realtime hardening completed on 2026-08-31.

## Completed changes

### Backend order webhook semantics

Merged PR #59 (`802de508196114b660f72648804aa110c6547ac7`).

Canonical order lifecycle:

- `PAID -> DELIVERED`: the business-order delivery service owns the `order.delivered` webhook and emits it only after the successful conditional state mutation.
- `DELIVERED -> COMPLETED`: the escrow-driven completion transition owns `order.completed` and emits it only when its conditional completion claim succeeds.
- A failed or losing concurrent transition does not emit a completion webhook.
- The premature controller-level `order.completed` dispatch was removed.
- No compatibility alias or duplicate webhook path was introduced.
- Focused regression tests cover successful delivery, failed delivery claims, successful escrow completion, and a losing concurrent completion claim.

Issue #49 was closed as completed after PR #59 merged.

### Flutter realtime consumer cleanup

Merged PR #33 (`862afd4fdb4c5f053fa77d6325a650a492a241a9`).

Removed proven-dead consumer aliases:

- `trade_completed`
- `order_location`
- `order_status`
- `order_eta`

Canonical active consumers remain `trade_update` and the colon-delimited order events.

## Escrow realtime contract

Backend PR #58 established post-commit lifecycle signals:

- `escrow_funded`
- `escrow_pending_settlement`
- `escrow_settled`

`escrow_terms_updated` is an existing real producer in `escrowController.updateTerms`; it must not be treated as a phantom event.

The financial rule is: authoritative DB mutation first, realtime emission second. Realtime delivery failure must not roll back a committed financial mutation.

## Group typing contract

The group socket service was traced directly. The client sends `group_typing` with `{ groupId, userId, isTyping }`. The Backend dynamically selects the event name:

- `isTyping === true` -> `group_typing_started`
- `isTyping === false` -> `group_typing_stopped`

The emitted payload is `{ groupId, userId }`.

Therefore `group_typing_started` and `group_typing_stopped` are canonical and require no convergence change. Static `.emit()`/`.on()` name matching must account for dynamically selected event names.

## Orphan-event audit rules

The original raw orphan count is an investigation queue, not a deletion queue. Multiple apparent orphans were proven legitimate after runtime-aware tracing:

- `friend_transfer_received` has a real Flutter consumer.
- queue events have real client consumers.
- group typing uses dynamic event names.

Any remaining orphan classification must resolve dynamic event names and distinguish:

1. missing producer;
2. missing consumer;
3. intentional server/internal/admin event;
4. future feature event;
5. genuinely dead producer/consumer.

No bulk deletion should occur from static name comparison alone.

## Financial/Admin boundary status

The Admin financial API boundary is substantially complete for the audited surfaces:

- disputes / War Room;
- escrow disputes;
- pagination;
- withdrawals contract and presentation;
- withdrawal approve/reject facade migration;
- payout settings;
- fee profiles;
- global settings.

The withdrawal UI now uses the Backend producer fields (`amount`, `payoutMethod`, `network`, `destination`, `status`, `createdAt`) rather than stale field names.

## Business Portal testing status

Business Portal PR #17 merged with Vitest/jsdom/Testing Library infrastructure and six route-rendering tests. Existing Vitest-based tests also execute. CI passed all 12 tests.

## Verification state

At the time of this record:

- all identified open PR work from this continuation is merged;
- Backend PR #59 is merged;
- Flutter PR #33 is merged;
- the known order/realtime correctness issues addressed in this phase are closed;
- no speculative realtime producer or consumer changes were introduced.

## Next engineering phase

The remaining work is broader production hardening rather than the original known realtime gaps:

1. complete the runtime-aware event inventory by domain;
2. add executable cross-repo contract tests for high-value financial/status events;
3. address the Backend E2E smoke teardown/schema-drift issue (#28) after re-verifying the current Prisma schema and test cleanup behavior;
4. address Admin typecheck debt in coherent batches rather than weakening the compiler gate;
5. audit Business Portal GitHub Actions availability and make CI a genuine merge gate when repository-level Actions availability permits;
6. perform the final duplicate-wrapper, stale-branch, abandoned-PR, and compatibility-layer audit.

## Engineering standard

Continue using:

`research -> implement -> check -> research -> implement -> test -> audit`

Do not optimize for PR count or green CI alone. Preserve atomic financial transitions, authorization, idempotency, balance/fee correctness, audit history, realtime timing, rollback semantics, and clean incremental diffs.