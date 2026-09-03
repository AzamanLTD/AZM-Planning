# AZAMAN Active Engineering Loop

**Purpose:** persistent continuation state for long-running engineering work. A new session MUST resume from this file when it is non-empty/active rather than restarting or asking the user to restate the task.

## Operating contract

1. Read `START_HERE.md`, `ROADMAP.md`, `CURRENT_STATE.md`, then this file.
2. Reconcile every referenced repo/PR/branch/SHA against GitHub before changing code.
3. Before creating a branch or PR, search existing branches, open PRs, recent merges and equivalent implementations/tests.
4. Execute the entire active work package: research → trace producers/consumers → inspect schema/authorization/state machine → implement → test → exact-head CI → diff audit → merge → verify `main` → update this file and `CURRENT_STATE.md`.
5. Do not stop merely because a defect was found, a patch was written, a local test passed, or a PR was opened.
6. If CI fails, diagnose and fix it; do not weaken tests/type checking to obtain green CI.
7. After a verified completion, immediately select the next unchecked P0 from `ROADMAP.md` unless genuinely blocked.
8. If blocked by missing access, an external dependency, or a decision that cannot safely be inferred, record the exact blocker and the smallest required user action. Otherwise continue autonomously.
9. Never duplicate completed work. If an existing implementation is partial, continue/consolidate it instead of creating a competing implementation.

## Current loop

**Status:** ACTIVE

**Work package:** Backend → Business OS → shift mutation/state boundary

**Objective:** Prevent generic shift PATCH from directly mutating lifecycle `status`; restore route permission parity for shift rotation; verify authorization semantics for attendance endpoints as part of the same boundary audit.

**Repo:** `AzamanLTD/AZM-backend`

**Branch:** `fix/business-os-shift-mutation-boundary`

**Base:** `main` at the verified pre-loop head after payroll PR #131 merge (`e81e32d83262a27c721ac51de8786b650f6be433`)

**Research already established:**
- `ShiftService.updateShift()` currently accepts `status` in its generic allowed-field list.
- Dedicated `clockIn`, `clockOut`, and `markNoShow` implement conditional lifecycle transitions and financial/employee side effects.
- `POST /shifts/rotation` currently lacks the same `shifts.create` permission middleware used by `POST /shifts`.
- Attendance routes delegate to service-level actor/business checks; their exact route-vs-service permission contract must be preserved or deliberately strengthened after audit.

**Completed in this loop:**
- Reconciled Backend PR #131: head `e0f63435f4688076146556c969f27c81601ccd30`, exact-head Azaman Test Suite run `33720636841` / run #558 passed, PR merged with merge SHA `e81e32d83262a27c721ac51de8786b650f6be433`.

**Remaining:**
- Implement and regression-test status mutation rejection.
- Add `shifts.create` to rotation route after permission-template compatibility check.
- Inspect existing test conventions and add focused compatible regression coverage.
- Run exact-head CI, audit, merge and verify main.
- Then continue immediately into the next P0: trace `updateAccruedWages()` consumers and begin dine-in `confirmAndPay` cross-repo contract audit.

**Do not redo:** payroll retry implementation in PR #131; it is merged and must be treated as completed unless current main verification disproves that state.

**Last verified backend main:** `e81e32d83262a27c721ac51de8786b650f6be433`

**Last verified Planning main:** `e61c070d94743be152ff46830eb984036305b540c8`

**Next exact action:** fetch current backend test structure and permission template, then implement the coherent shift boundary batch on the branch above.

## Completion record

A loop may be marked `COMPLETE` only after the implementation is on `main`, exact-head CI is green for the final feature head, the merge/main SHA is verified, and `CURRENT_STATE.md` records evidence and residual risk.
