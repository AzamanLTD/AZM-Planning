# 2026-09-14 Business OS shift scheduling/state integrity — verified remediation

## Finding
Three `ShiftService` mutations were raceable under concurrency:
1. `createShift` performed a read-then-write conflict check — two concurrent calls could both observe a conflict-free schedule and commit overlapping active shifts.
2. `updateShift` had no overlap protection at all — time changes could create/update overlaps against other active shifts of the same employee.
3. `deleteShift` did a stale pre-read + delete-by-id — a concurrent `clockIn()` could activate the shift between the check and the delete, and the delete would remove an active shift.

## Remediation implemented
`AZM-backend` PR #247 (squash `a6b025a` on main):
- `createShift`: conflict check + insert in ONE transaction, serialized by a transaction-scoped PostgreSQL advisory lock scoped to the employee schedule namespace (`pg_advisory_xact_lock(hashtext('business_shift_schedule:<employeeId>'))`, acquired through `tx`, held until commit — same pattern as businessTaxPresetService / orderTrackingMutationSafeService). `businessProfileId` added to the conflict predicate; existing tenant/inactive-employee checks and response shape preserved.
- `updateShift`: target loaded inside the transaction by `id + businessProfileId`; when `startTime`/`endTime` change, other active/scheduled shifts of the SAME employee/business (excluding the target) are checked under the SAME advisory lock, rejecting any overlap with the existing error. Non-time updates skip the overlap work entirely. Caller-supplied `status` remains rejected; clockIn/clockOut/markNoShow remain the only lifecycle transitions.
- `deleteShift`: guarded `deleteMany` (`id + businessProfileId + status NOT CLOCKED_IN/LATE`) inside a transaction; `count !== 1` re-reads and throws the exact existing errors (`Shift not found.` / `Cannot delete an active shift.`). The conditional DELETE is the authority against clockIn — clockIn does not participate in the scheduling lock namespace, so the guarded delete closes the stale-read race without weakening clockIn (whose guarded SCHEDULED-only transition independently refuses in every interleaving).

No new state machine, no new ledger, no schema change.

## Regression coverage
Added `__tests__/shift-scheduling-integrity.test.js` (real PostgreSQL, DB-gated, 7 cases): concurrent overlapping `createShift` → exactly one winner; concurrent non-overlapping `createShift` → both succeed; concurrent time-changing `updateShift` cannot commit an overlapping schedule (each update valid alone, results conflict — the race the lock closes); update-to-overlap rejected with zero mutation; delete-vs-clockIn race ends consistent (exactly one winner, never a deleted active shift); business isolation (another business scope cannot mutate); generic status mutation still rejected. `shiftService.business-scope.test.js` updated for the new transactional `updateShift` call shape.

## Verification
- PR: `AzamanLTD/AZM-backend#247`
- Exact CI head: `9f6c4660024f31e47a39c6aae2537bbaa7feae9b`
- CI green: **1212/1212 tests**, route registry verification PASS, database backup/restore recovery drill clean.
- Merged to backend main as squash `a6b025af635d068eee844a2f06a4ac94baf5adff`.

## Critical platform finding (follow-up required)
`pg_advisory_xact_lock()` returns a **void** column that the Prisma query engine cannot deserialize in either `$queryRaw` form — CI proved this twice on real PostgreSQL. Both existing advisory-lock patterns in the codebase are latent CI failures that only pass because their tests never execute the lock path against a real database:
- tagged-template form: `orderTrackingMutationSafeService`, `storefrontDraftMutationSafeService` (+ publish counterpart)
- `$queryRawUnsafe` plain form: `businessTaxPresetService`

The working shape (shipped here) is:
```js
await tx.$queryRawUnsafe('SELECT 1 AS locked FROM pg_advisory_xact_lock(hashtext($1))', key);
```
**Follow-up:** apply this one-line fix to the four services above and add a DB-gated test that actually executes each lock path against real PostgreSQL.

## Broader audit notes
`clockIn` already used a guarded SCHEDULED-only `updateMany` transition (atomic, correct) and was deliberately left untouched. Shift swap approve already runs as one Serializable transaction with retry; swap request/claim already verify business scope and employee identity. No additional P0 was opened from those paths during this loop.
