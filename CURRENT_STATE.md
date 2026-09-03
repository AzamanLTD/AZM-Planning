# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file → `ACTIVE_LOOP.md` → `EXECUTION_LEDGER.json` before engineering. Historical reasoning belongs in Git history/archive, not here.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `a0876b2f61d5bc73acb1a1d76368e019d079fe82` after PR #141 | POS #145 and kiosk #144 are fresh current-main PRs; finish exact-head CI/review before merge |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes verified PR #78 failure propagation | Continue dine-in client/server convergence audit; verify end-to-end payment/retry semantics |
| `AZM-Planning` | Persistent continuation files exist on main | Reconcile after every substantial verified batch |

**Never rely on historical SHAs after another session without reconciling them against GitHub.**

## 2. Planning-brain consolidation — VERIFIED

PR #24 established the canonical engineering brain. Persistent continuation files are present: `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`. `ROADMAP.md` remains the active execution authority.

## 3. Verified backend hardening

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions race-safe in PR #127.
- Time-off approval/rejection tenant-scoped and concurrency-safe in PR #126.
- Payroll bounded P2034 retry merged/verified in PR #131.
- Shift generic PATCH status boundary merged/verified in PR #132.
- Dine-in settlement/finalization/idempotency hardening merged in PR #135.
- KYB gate fail-closed hardening merged in PR #136.
- Business OS finance canonical routing/runtime repair merged in PR #137.
- Inventory restock atomicity merged in PR #141 as `a0876b2f61d5bc73acb1a1d76368e019d079fe82`.
- The formerly failing dine-in customer-order test was corrected in merged PR #143 on the pre-inventory main lineage; subsequent exact-head runs for current mutation work are green.

## 4. Current mutation batch

### POS — current replacement PR #145

- PR #140 is closed/superseded; PR #138 was already superseded.
- PR #145 is open on `fix/business-os-pos-atomicity-v3`, head `c01ab35692f7cf237387ecf00d5454ad748a2c57`, targeting main `a0876b2f61d5bc73acb1a1d76368e019d079fe82`.
- The implementation server-derives catalog prices, validates integer quantities, supports CASH/AZM/SPLIT, conditionally debits AZM, writes `BusinessOrderItem` rows, commits order/line-items/ledger atomically with Serializable retry, preserves tenant-scoped idempotent replay, and now consumes tracked `BusinessProduct.stockQty` plus restaurant `RecipeIngredient` inventory in the same transaction.
- Inventory consumption is guarded by transaction-time stock predicates to prevent overselling under concurrency; shared ingredients are aggregated per order before decrement.
- Remaining contract risk: the 2.5% tax is legacy-compatible but not yet proven to be the final tax authority; reconcile against `BusinessTaxPreset`/invoice tax semantics before production confidence. Also continue location/table integrity and idempotency-payload semantics review.
- Exact-head CI run is active: `33776598303` on head `c01ab356...`; do not merge until it completes green and the final diff is re-reviewed.

### Inventory — VERIFIED

- PR #141 was merged as `a0876b2f61d5bc73acb1a1d76368e019d079fe82`.
- Canonical restock route/service is now on main with business scoping, positive quantity/non-negative cost validation, and atomic stock increment + `SUPPLIES` expense ledger write.
- The previous CI diagnosis of zero-step runner failure is superseded. Current public repositories are executing the full release workflow successfully.

### Kiosk — current replacement PR #144

- PR #142 is closed/superseded because it was based on the pre-inventory main head.
- PR #144 is open on `fix/business-os-kiosk-capability-v2`, head `4fca0d65a416b85cefa22ec4b15256b5a6cff25d`, targeting current main `a0876b2f...`.
- Capability signing/verification is isolated; clock-in/out enforce tenant/employee/user/shift binding and location binding; PIN auth validates that a supplied location belongs to the business.
- This replacement also adds a strict second-layer rate limit to the public PIN challenge (`10` attempts per `15` minutes, successful attempts excluded) on top of the existing route-level limiter.
- Exact-head CI run `33776303876` is currently in progress at the Jest test stage; do not merge until it completes green.

## 5. CI/release-gate evidence

The backend repositories are now public and GitHub Actions is executing the canonical workflow normally. Recent successful exact-head runs include POS, inventory and kiosk predecessor heads; the current replacement PRs are also executing rather than failing at runner startup. The release gate remains unchanged: tests and the database recovery drill must pass at the exact head before merge.

The earlier failing inventory run `33749730364` was a real Jest failure on `dine-in-customer-order-settlement.test.js` (128 suites passed, 1 failed; 898 tests passed, 1 failed), not a mysterious runner failure. That missing mock was corrected in PR #143. Subsequent mutation runs `33774674487`/`617`, `33774689580`/`618`, and `33774699365`/`619` executed the complete pipeline successfully.

## 6. Frontend dine-in payment — VERIFIED item only

PR #78 was verified with Flutter Quality and merged. `DineInTabNotifier.payTab()` now records failure state and rethrows, preventing false success UI.

This closes frontend failure-truthfulness only. The broader `confirmAndPay` cross-repo contract remains open for payment authority, idempotency, tab closure, tips, realtime and reconciliation.

## 7. Active P0 queue

1. Finish exact-head CI + final diff audit + merge for PR #144 and PR #145.
2. Finish POS tax-authority, location/table, inventory and replay contract review; add authoritative tax tests rather than preserving legacy behavior by assumption.
3. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter.
4. Trace every `updateAccruedWages()` consumer/history before removal or restriction; `ShiftService.clockOut()` already performs accrued-wage mutation inside its own transaction.
5. Add Admin financial mutation coverage and correct optimistic pending-queue behavior.
6. Complete tenant/state/realtime matrices, then production-ops/load/red-team waves.

## 8. Open risks

- POS legacy 2.5% tax is not yet proven to be the canonical BusinessTaxPreset/invoice tax authority.
- POS location/table integrity and customer identity semantics require final producer/consumer evidence.
- Idempotency keys are unique, but a complete payload-binding policy still needs explicit contract evidence.
- Kiosk PIN brute-force is now locally rate-limited, but distributed deployment and enumeration/timing behavior still need validation.
- Dine-in end-to-end reconciliation and realtime remain open.
- Deployment/staging, migration rollback, secret lifecycle, realtime recovery, Admin mutation coverage and load-testing evidence remain open.

## 9. Session update protocol

Every substantial engineering batch must update this file plus `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` with exact repo/PR/head, CI evidence, residual risk and next action. Never mark work VERIFIED from discussion alone.

**2026-09-03 live-main reconciliation:** public-repository Actions execution is now demonstrably working. Inventory PR #141 is merged to backend main. Old POS #140 and kiosk #142 were closed as stale-base duplicates. Fresh current-main replacements are #145 (POS with atomic inventory consumption) and #144 (kiosk capability with PIN rate limiting). Their exact-head runs are active; merge only after green completion and final diff review.
