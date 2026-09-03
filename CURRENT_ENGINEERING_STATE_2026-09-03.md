# AZAMAN — Current Engineering State — 2026-09-03

This snapshot supersedes prior dated continuation notes for live implementation status. It records only repository state verified during the current engineering loop.

## Production merges completed in this loop

### Backend — `AzamanLTD/AZM-backend`

- PR #126 merged: Business OS time-off approval/rejection is business-scoped and concurrency-safe. Approval/rejection now derives scope from the approver, blocks self-approval, and uses conditional `updateMany` against `PENDING` state.
- PR #127 merged: Business OS shift attendance and swap transitions are race-safe. `clockIn`, `clockOut`, and `markNoShow` use conditional state transitions inside transactions; employee aggregates update only after a successful transition; swap claims are conditional; `LATE` shifts are included in team-on-duty; active late shifts cannot be deleted.
- PR #128 merged as `e6497cfbd3925a7f7e5a2e83f2d59e211037e52e`: Serializable shift-swap approval retries only Prisma `P2034` conflicts, with bounded backoff and focused regression coverage. Exact-head Azaman Test Suite run passed tests and the DB backup/restore drill before merge.
- PR #129 merged after exact-head CI verified the worker dashboard attendance consumer is consistent with shift state: `CLOCKED_IN` and `LATE` are both treated as active for current shift and team-on-duty views. The first fixture attempt failed because the dashboard performs three sequential shift reads; the corrected regression fixture passed the full suite and recovery drill.
- PR #130 merged as `68bd1d51a28c61f31e556f616c488b2d7ce631a2`: EWA withdrawals now retry only transient Prisma `P2034` serialization conflicts with bounded backoff. The EWA boundary also enforces business scope and request-context authorization: owners/admins may manage, workers may withdraw for themselves, and other employees require `ewa.manage`. EWA eligibility/history are tenant-scoped when request context is present. Exact-head Azaman Test Suite run #556 passed all tests and the DB backup/restore drill before merge.

Earlier September merges still in production include the authoritative Business OS revenue series (#123), notification deduplication scoped by trade (#124), and the existing bounded feedback Serializable retry (#121).

### Admin Portal — `AzamanLTD/AZM-adminPortal`

- PR #94 merged as `00c2206dd0b5f1cac6c3eef0bd5954de4726dbd8`.
- Corrected the dashboard `Active Vendors` KPI navigation from `/businesses` to `/users`, matching the `/api/admin/stats` metric's vendor-user semantics. The merged diff is exactly one line and CI #236 passed lint, typecheck, and build.

### Flutter — `AzamanLTD/AZM-frontend`

- PR #77 merged as `b625c7da9b17e170bae40182ddaf767ab717f872`.
- Foreground socket notifications now use the existing accessible `InAppPushBanner` rather than raw SnackBars. The implementation preserves haptic feedback and notification-count updates and forwards socket `action` payloads into the existing notification action router. Flutter Quality run #313 passed analyzer, the full test-with-coverage stage, and coverage upload.

### Business Portal — `AzamanLTD/AZM-businessPortal`

Latest verified production merge remains PR #43 (`ee9bc4630b947c84c60f330f72d282c7e5e02ebc`), which switched invoice KPIs to the authoritative `invoiceStats()` endpoint. No new Business Portal production change was required in this loop.

## Current open work

- Backend PR #129 is merged.
- Backend PR #130 is merged.
- Frontend PR #77 is merged.
- Admin PR #94 is merged.
- Planning PR #23 is closed as stale; current engineering state is maintained directly on `main` in this file.

## Next engineering priorities

1. Audit `ShiftService.updateShift()` because its generic `status` field can mutate attendance state outside the accounting invariants owned by the dedicated clock-in/out/no-show methods. Confirm every route/UI consumer before changing behavior.
2. Continue employee/EWA authorization and read-scope audit, including worker-facing EWA eligibility/history and any legacy direct service callers, without weakening compatibility where no HTTP context exists.
3. Audit payroll disbursement and adjacent employee accrual paths for Serializable contention, duplicate requests, and obsolete raw-ID mutation helpers. Prefer evidence-based changes over adding retries indiscriminately.
4. Continue Business Portal runtime/API testing against actual production pages and query/mutation consumers.
5. Build the cross-repo event/status contract matrix from actual Backend producers through Admin, Business, and Flutter consumers, including deduplication and reconnect behavior.
6. Complete the marketplace dead-code/duplication audit using reference and route evidence before deleting anything.
7. Finish branch/PR reconciliation only after verifying exact current refs; do not resurrect stale branches or treat historical merged branches as active work.

## Engineering protocol

Every meaningful change continues to follow:

`research → trace producers/consumers → implement minimally → test → exact-head CI → audit final diff`

CI remains a gate. For financial or stateful mutations, review authorization, business scoping, transaction boundaries, conditional state transitions, idempotency, balance/ledger/history effects, notifications, realtime timing, rollback behavior, and HTTP error semantics.
