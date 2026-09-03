# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file before engineering. This file is deliberately short; historical reasoning is preserved in Git history/archive rather than mixed into active instructions.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `924807b3742f30f929479d46bda96d9660b61f2d` after PR #137 | POS/inventory atomicity remain unmerged; exact-head CI execution is currently unreliable |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes PR #77 notification banner hardening | Re-verify payment-failure semantic fix claim; full journey authority/realtime audit; dine-in contract first |
| `AZM-Planning` | This reconciliation is being prepared on a dedicated branch; main is unchanged until the PR is merged | Keep continuation state synchronized after every substantial batch |

**Verify current application SHAs against GitHub before relying on them if another session has changed the repos.**

## 2. Planning-brain consolidation — VERIFIED

PR #24 (`docs: establish canonical engineering brain`) was merged at `da489d71fbe76f411494315c7e20f2c85d1bbc4b` after an exact-head merge. The active planning surface is now:

- `START_HERE.md` — mandatory continuation entrypoint;
- `ROADMAP.md` — single active execution/priority authority;
- `CURRENT_STATE.md` — live state/evidence ledger;
- `ARCHITECTURE.md` — system/authority map;
- `CONTRACTS.md` — cross-repo contracts;
- `FINANCIAL_INVARIANTS.md` — money safety;
- `SECURITY_BOUNDARIES.md` — authorization/tenant/actor boundaries;
- `STATE_MACHINES.md` — lifecycle rules;
- `RELEASE_CHECKLIST.md` — production gate;
- `REPO_GUIDE.md` — continuation protocol;
- `ACTIVE_LOOP.md` / `EXECUTION_LEDGER.json` — persistent continuation state;
- `archive/` — historical policy/index plus preserved Git history.

Superseded dated root planning files were removed from the active surface; their exact historical content remains recoverable from Git history. This is intentional: fewer competing instructions, not loss of provenance.

## 3. Verified backend hardening and current batch

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions made race-safe in PR #127.
- Time-off approval/rejection made tenant-scoped and concurrency-safe in PR #126.
- Payroll disbursement bounded P2034 retry merged/verified in PR #131.
- Shift generic PATCH status boundary merged/verified in PR #132.
- Business OS finance canonical routing/runtime repair merged in PR #137 at main `924807b3742f30f929479d46dba96d9660b61f2d`.
- Dine-in settlement/finalization/idempotency hardening merged in PR #135 at `fc24fcc61800e86cad9657b91b59c8d24e93a8ef`; cross-repo reconciliation remains active.
- KYB gate fail-closed hardening merged in PR #136.

### Current POS/inventory batch — CI still blocked, implementation refined

- Former POS PR #138 is **closed and superseded**, not merged. It accumulated stale-base/diagnostic churn and is retained only for audit history.
- POS replacement PR #140 (`fix(pos): atomic Business OS POS settlement (current main)`) is open on `fix/business-os-pos-atomicity-v2`.
- Current POS head is `b96bfa64491fb0e8790dcc6c4b1720237e17976c`.
- POS now server-derives active business catalog pricing, enforces integer quantity bounds, supports CASH/AZM/SPLIT, performs conditional AZM debit, writes `BusinessOrderItem` line items, and commits the order/line items/ledger atomically under Serializable retry.
- POS idempotent replay was refined to resolve **before catalog validation**. This preserves offline retry semantics when a product has subsequently been disabled or removed, while still rejecting cross-business idempotency-key reuse.
- POS exact-head CI is not currently green/proven: no pull-request run exists for head `b96bfa64491fb0e8790dcc6c4b1720237e17976c`. Earlier equivalent PR attempts produced both normal test-stage failures and separate job-startup failures. Do not merge until an exact-head full `Azaman Test Suite` run succeeds.
- POS contract audit still required before release: tax authority, ledger currency/unit semantics, customer/payment identity, inventory effects, location/table integrity, and replay semantics.

- Inventory replacement PR #141 (`fix(inventory): atomic restaurant restock on current main`) is open on `fix/business-os-inventory-restock-atomicity-v2`.
- Current inventory head is `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`.
- Inventory service now scopes items to the business, validates quantity/cost, and updates stock plus signed SUPPLIES expense in one transaction; the test suite also covers ledger failure propagation.
- An accidental/temporary diagnostic workflow was introduced during CI investigation and then removed; current PR #141 is back to the four intended implementation/test/route files.
- Exact-head CI remains unresolved: recent runs `33746032891` (#604), `33746106832` (#605), `33746223780` (#606), and `33746364784` (#607) failed before executable job steps were recorded. Run #607 was explicitly rerun and again produced a zero-step failure. This is not evidence that the implementation itself passed; it is evidence of the current CI execution problem.
- PR #139 is the older duplicate inventory branch and should be closed deliberately as superseded once the replacement path is stable; do not merge both implementations.

### CI/release-gate blocker

The canonical Backend workflow remains intact; the known-good PR #137 run `33732480681` executed setup, tests and the DB recovery drill successfully. Current newly written PR heads can instead fail in GitHub before any job step is exposed, while earlier attempts on the same work also had genuine test-stage failures. The release gate must remain unchanged. The correct response is to obtain reliable exact-head execution/evidence, not to weaken CI or merge on assumptions.

## 4. Weakest/highest-risk surfaces

**CI/release gating:** current PR integration is not producing reliable exact-head evidence for the latest POS/inventory heads.

**Admin Portal:** small test surface relative to force-release/cancel escrow, withdrawal approval/rejection, fee controls and War Room actions. This is a production blocker.

**Dine-in:** first cross-repo contract audit because it crosses Flutter, Backend and Business Portal, moves real money and has a finalize/payment race.

**Production operations:** staging/deployment, rollback, secret rotation, populated-data migration safety, monitoring and load evidence are not yet proven.

**Engineering hygiene:** stale/duplicate branches and open PRs must be reconciled before overlapping implementation is started.

## 5. First implementation queue

1. Resolve/obtain reliable exact-head CI execution for POS #140 and inventory #141 without weakening the release gate; merge only after exact-head green evidence.
2. Finish POS contract audit against legacy producers/consumers: tax authority, line-item semantics, inventory effects, payment/customer identity, ledger units, and idempotency replay behavior.
3. Close/supersede duplicate inventory PR #139 deliberately; then independently gate #141 against the latest main.
4. Finish kiosk capability hardening and exact-head verification.
5. Trace every `updateAccruedWages()` consumer/history and remove or restrict only after proving redundancy/safety.
6. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter, including concurrent payment, idempotency, tab closure, tip authority, realtime and client convergence.
7. Add Admin financial mutation tests: withdrawals, escrow disputes, fee engine, War Room and dashboard/optimistic state.
8. Complete tenant/state matrices for remaining Business OS and commerce objects.
9. Implement/verify realtime missed-event reconciliation.
10. Move into production-ops, load and red-team waves only after the investor-critical mutations have clean release evidence.

## 6. Open risks

- Exact-head CI is still not proven for the latest POS/inventory heads.
- POS tax authority is unresolved; the implementation still has a 2.5% derived tax pending contract evidence.
- POS inventory-decrement semantics and location/table/customer identity require producer/consumer verification.
- Inventory restock has no structural idempotency key yet; adding one requires schema/contract authority before implementation.
- Deployment/staging process is not yet proven by current evidence.
- Code + Prisma migration rollback/recovery has not been rehearsed.
- Secret lifecycle/rotation is not yet proven.
- Realtime missed-event recovery is not yet fully demonstrated.
- Admin critical mutation coverage remains thin.
- Populated-production migration safety is not yet demonstrated.
- Load-testing evidence is absent at the target minimums.

## 7. Session update protocol

Every substantial engineering batch must update this file with: change, repo, PR/commit, exact tests/CI evidence, residual risk and next exact action. Never mark work VERIFIED from discussion alone.

**2026-09-03 reconciliation:** POS PR #140 advanced to `b96bfa64491fb0e8790dcc6c4b1720237e17976c` with replay-safe idempotency ordering and focused coverage. Inventory PR #141 advanced to `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5` after removing temporary diagnostic workflow churn and retaining transaction/ledger regression coverage. Exact-head CI remains blocked by GitHub job-startup failures/no-step runs; neither change is marked verified or merged.
