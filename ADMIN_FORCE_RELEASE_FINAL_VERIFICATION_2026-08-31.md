# Admin Force Release — Final Verification — 2026-08-31

## Status

Issue `AzamanLTD/AZM-backend#48` is completed and closed.

Backend atomicity merged in PR #57:
- merge SHA: `ca7f7095302d89f0eea1fec3b36d0069aca9768a`
- `completeTrade()` accepts `DISPUTED` only with explicit `adminOverride`;
- normal callers remain `PAID -> COMPLETED`;
- Admin passes `releasedByUserId` through for audit/profit/history semantics;
- the old separate `DISPUTED -> PAID` claim is gone;
- `TRADE_ALREADY_FINALIZED` maps to HTTP 409;
- `forceCancel` was intentionally untouched.

## Admin Portal convergence

Admin Portal PR #17 merged:
- merge SHA: `444af13f9e22b5adc67fe3142cf32275045079fb`
- CI run #39 passed before the final ready-workflow cleanup commit;
- final PR diff remained exactly one application file: `src/pages/WarRoom.jsx`, 10 additions / 1 deletion;
- the `forceRelease` mutation preserves optimistic rollback;
- HTTP 409 now invalidates/refetches the active `admin/disputes` query and explicitly reports that another admin won the settlement race;
- no false success toast is produced for the 409 path;
- `forceCancel` and normal dispute resolution were not changed.

The temporary workflow used to make the PR ready was removed before merge and is not part of the final application diff.

## Business Portal

The Business Portal's canonical realtime bridge listens for `escrow_*` lifecycle events and invalidates authoritative order projections. Its Business Notification service explicitly resolves an escrow to a `BusinessOrder` and is a no-op when no business order exists.

The Admin force-release path is a P2P trade path. It does not manufacture a BusinessOrder notification or create a second Business Portal-specific settlement state. Therefore no additional Business Portal listener was added.

This is intentional: do not invent a business-order event for a P2P trade. If a future product flow links Admin force-release to a BusinessOrder, the existing `bizNotificationService.notifyOrderEvent()` chokepoint is the correct integration point.

## Flutter

Flutter `SocketService` on `main` (merged PR #31, SHA `b996e65e015ae4670e74e94f33f3c26b2f6dfa34`) registers `escrow_refunded` through the existing escrow event registry. It also already consumes `new_notification` through `NotificationProvider`.

Admin force-release uses the existing Backend `NotificationService`-style user-room notification contract for participant notification; no second socket connection or duplicate notification architecture was introduced.

## Contract decision

For Admin force-release, financial state is authoritative in the Backend transaction. Socket events are convergence signals only. The Admin 409 path deliberately refetches rather than trusting an optimistic state. Business Portal does not receive a synthetic P2P business-order event. Flutter receives participant notification through the existing notification channel.

## Remaining infrastructure item

`main` branch protection remains a separate repository-administration task. The available GitHub connector can read protection state but does not expose the required branch-protection write endpoint. Do not fake protection through application CI.
