# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file before engineering. This file is deliberately short; historical reasoning is preserved in Git history/archive rather than mixed into active instructions.

## 1. Production baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `924807b3742f30f929479d46bda96d9660b61f2d` after PR #137 | POS/inventory/kiosk changes remain unmerged; exact-head CI execution is unreliable |
| `AZM-adminPortal` | Main `2a4958faac6e5af3066972cfe7634abef800c474` after PR #95 | Critical financial mutation tests; optimistic state; control-plane visibility |
| `AZM-businessPortal` | Main `5ab6cd64665608a4794b93b924d330a427a00b26` after PR #43 | KPI authority; Finance/Invoices/Orders tests; Blueprint/runtime contract |
| `AZM-frontend` | Main includes PR #77 notification banner hardening | Re-verify payment-failure semantic fix claim; full journey authority/realtime audit; dine-in contract first |
| `AZM-Planning` | Main currently contains the prior reconciliation PR; this kiosk/CI reconciliation is on a dedicated branch until merged | Keep continuation state synchronized after every substantial batch |

**Verify current application SHAs against GitHub before relying on them if another session has changed the repos.**

## 2. Planning-brain consolidation — VERIFIED

PR #24 (`docs: establish canonical engineering brain`) was merged at `da489d71fbe76f411494315c7e20f2c85d1bbc4b` after an exact-head merge. The active planning surface is now:

- `START_HERE.md` — mandatory continuation entrypoint;
- `ROADMAP.md` — single active execution/priority authority;
- `CURRENT_STATE.md` — live state/evidence ledger;
- `ARCHITECTURE.md` — system/authority map;
- `CONTRACTS.md` — cross-repo contracts;
- `FINANCIAL_INVARIANTS.md` — money safety;
- `SECURITY_BOUNDARIES.md` — authorization, tenant and actor boundaries;
- `STATE_MACHINES.md` — lifecycle rules;
- `RELEASE_CHECKLIST.md` — production gate;
- `REPO_GUIDE.md` — continuation protocol;
- `ACTIVE_LOOP.md` / `EXECUTION_LEDGER.json` — persistent continuation state;
- `archive/` — historical policy/index plus preserved Git history.

## 3. Verified backend hardening and current batch

- EWA serialization retry/scoping hardening merged in PR #130.
- Shift attendance/swap transitions made race-safe in PR #127.
- Time-off approval/rejection made tenant-scoped and concurrency-safe in PR #126.
- Payroll disbursement bounded P2034 retry merged/verified in PR #131.
- Shift generic PATCH status boundary merged/verified in PR #132.
- Business OS finance canonical routing/runtime repair merged in PR #137 at main `924807b3742f30f929479d46bda96d9660b61f2d`.
- Dine-in settlement/finalization/idempotency hardening merged in PR #135 at `fc24fcc61800e86cad9657b91b59c8d24e93a8ef`; cross-repo reconciliation remains active.
- KYB gate fail-closed hardening merged in PR #136.

### POS — current replacement PR #140

- PR #138 is closed/superseded and was never merged.
- PR #140 (`fix(pos): atomic Business OS POS settlement (current main)`) is open on `fix/business-os-pos-atomicity-v2` at head `b96bfa64491fb0e8790dcc6c4b1720237e17976c`.
- Server-authoritative catalog pricing, integer quantity validation, CASH/AZM/SPLIT, conditional AZM debit, authoritative `BusinessOrderItem` persistence, and order/line-items/ledger transactionality are implemented.
- Idempotent replay now resolves before catalog validation, preserving safe offline replay after catalog changes while still enforcing tenant ownership of keys.
- Legacy producer tracing established that the existing POS `/pos/order` also applies the same 2.5% tax and defaults `customerId` to the authenticated actor when none is supplied. This removes those two items as unexplained deviations; remaining contract audit concerns are ledger units, inventory effects, location/table integrity, and replay response semantics.
- There is still no exact-head green release run attached to `b96bfa...`; do not merge until a full exact-head `Azaman Test Suite` succeeds.

### Inventory — current replacement PR #141

- Older duplicate PR #139 is now **closed and not merged**; PR #141 is the sole current inventory path.
- PR #141 (`fix(inventory): atomic restaurant restock on current main`) is open on `fix/business-os-inventory-restock-atomicity-v2` at head `ae8311f44fb8aef5ceac90ed947d9ab5c37b01d5`.
- Stock mutation and signed SUPPLIES expense are in one transaction; quantity/cost/business scoping is validated; focused coverage includes ledger-failure propagation.
- Temporary diagnostic workflow churn was removed.
- Exact-head runs `33746032891` (#604), `33746106832` (#605), `33746223780` (#606), and `33746364784` (#607, rerun attempt 2) all failed before executable job steps were exposed. Do not interpret these as code passes or use them to bypass the gate.

### Kiosk — current PR #142

- PR #142 (`fix(kiosk): enforce scoped clock capability and location binding`) is open on `fix/business-os-kiosk-capability` at head `40c78279c87e818071f0b9317149e96904ddc3eb`.
- Capability signing/verification was isolated into `services/businessOS/kioskCapability.js`.
- Clock-in/out now revalidate active employee identity and the target shift against capability tenant/employee/user binding; a location-bound capability must match the shift location.
- PIN authentication validates a supplied location belongs to the business before issuing the capability.
- Focused tests cover capability expiry, scope, tenant, employee and location binding.
- Exact-head CI run `33746899789` (#608) failed before executable job steps were exposed, matching the current cross-PR runner/job-startup failure pattern. Do not merge until exact-head full CI is green.

### CI/release-gate blocker

The canonical Backend workflow remains unchanged. Known-good PR #137 run `33732480681` executed setup, tests and DB recovery successfully. New PR heads can instead fail before executable job steps are exposed, and earlier attempts on the same POS/inventory batch also reached the real test stage and failed. The release gate must remain unchanged; diagnose GitHub Actions execution reliability rather than weakening tests or merging on assumptions.

## 4. Weakest/highest-risk surfaces

**CI/release gating:** current PR integration is not producing reliable exact-head evidence for the active Backend mutation branches.

**Admin Portal:** high-risk financial mutation test coverage remains thin.

**Dine-in:** still the first cross-repo contract audit because it crosses Flutter, Backend and Business Portal, moves money and has a finalize/payment race.

**Production operations:** staging/deployment, rollback, secret rotation, populated-data migration safety, monitoring and load evidence are not yet proven.

## 5. First implementation queue

1. Resolve reliable exact-head CI execution for POS #140, inventory #141 and kiosk #142 without weakening the release gate; merge only after exact-head green evidence.
2. Finish the POS contract audit: inventory decrement, ledger units, location/table integrity, and replay response contract.
3. Finish kiosk abuse-resistance/rate-limit review and verify exact-head CI.
4. Trace every `updateAccruedWages()` consumer/history before removal or restriction.
5. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter.
6. Add Admin financial mutation tests and correct optimistic pending-queue behavior.
7. Complete tenant/state/realtime matrices, then production-ops/load/red-team waves.

## 6. Open risks

- Exact-head CI remains unproven for POS #140, inventory #141 and kiosk #142 because fresh runs fail before executable steps.
- POS inventory-decrement semantics and location/table integrity still require producer/consumer evidence.
- Inventory restock has no structural idempotency key; schema/contract authority is required before adding one.
- Kiosk PIN enumeration/rate limiting should be strengthened beyond the existing general route limiter if the kiosk is exposed broadly.
- Deployment/staging, migration rollback, secret lifecycle, realtime recovery, Admin critical mutation coverage and load-testing evidence remain open.

## 7. Session update protocol

Every substantial engineering batch must update this file with change, repo, PR/commit, exact tests/CI evidence, residual risk and next exact action. Never mark work VERIFIED from discussion alone.

**2026-09-03 kiosk/CI reconciliation:** duplicate inventory PR #139 was deliberately closed. POS #140 advanced to `b96bfa...` with replay-safe idempotency ordering and legacy-contract verification. Kiosk PR #142 advanced to `40c782...` with isolated capability helpers, expiry/scope tests and enforced shift/location binding. Exact-head runs remain blocked by repeatable zero-step GitHub Actions failures. None of these active PRs is marked verified or merged.
