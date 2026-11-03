# AZAMAN Engineering Brain — START HERE

**Status:** ACTIVE — PRIMARY CONTINUATION ENTRYPOINT  
**Last verified:** 2026-09-05 UTC  
**Repositories:** `AZM-backend`, `AZM-adminPortal`, `AZM-businessPortal`, `AZM-frontend`, `AZM-Planning`

> Any agent continuing AZAMAN engineering MUST start here, then read `ROADMAP.md`, `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`. Do not restart from historical session journals.

## Mission

Bring AZAMAN to production-ready correctness by establishing who owns truth, who may mutate it, how state transitions, how concurrency behaves, and how every client converges after failure/realtime loss.

## Founder-approved financial programme — 2026-11-03

**This is the active founder-directed financial product programme. Read the full requirements and both agent work orders before starting implementation:**

- **Canonical programme / cross-chat continuation record:** [AZM-Planning issue #78 — Two-wallet GHS/USDC financial core and Moolre bank payout programme](https://github.com/AzamanLTD/AZM-Planning/issues/78)
- **Backend Agent A work order:** [AZM-backend issue #389 — Two-wallet financial core, native-GHS withdrawals, funding planner and bank beneficiaries](https://github.com/AzamanLTD/AZM-backend/issues/389)
- **Backend Agent B work order:** [AZM-backend issue #390 — Moolre Ghana bank payout provider adapter, validation and status contract](https://github.com/AzamanLTD/AZM-backend/issues/390)

Founder direction approved:
- GHS and USDC are distinct customer wallet assets and liabilities; display FX is never spendable balance or transaction authority.
- GHS mobile-money and bank payouts use GHS; crypto withdrawals use USDC.
- Insufficient funds may be covered by an explicitly quoted, consented shortfall conversion from the other wallet. Never silently sweep or convert.
- Saved bank beneficiaries must be securely stored and available in System Accounts and the relevant payout selector.
- Moolre remains the primary fiat provider. Bank withdrawals are the first new bank rail; bank collections are a later Add Cash phase.
- The Business Portal Finance model and Admin payout/reconciliation views must become asset-aware.
- Go may later perform bounded asynchronous work, but Node remains the only final financial authority.

**Parallel ownership:** Agent A owns financial accounting, native-GHS settlement/withdrawal, funding planner, saved-bank beneficiary persistence and financial orchestration. Agent B owns the Moolre bank-provider adapter and adapter-boundary tests/docs only. Agent B does not edit ledger/schema/routes; Agent A does not edit Agent B's provider adapter. The provider is wired into the financial path only in a later reviewed integration slice.

**Hard boundary:** these issues authorise implementation work in open PRs only. They do not authorise merges, deployments, production configuration/data changes, real provider transactions, or turning on live-money feature flags. Exact-head CI and independent review remain mandatory. Do not weaken the audit/test gates.

Refresh current main SHAs, PR heads/bases, branch ancestry and CI before any code change. Existing #369/#381/#382 native-GHS work and #379/#380 payout-destination/dust chain must be inspected and composed, not duplicated. Planning PR #75 is historical design input, not an implementation branch or proof of current code.

## Current priority

Current verified main baselines are Backend `66b39b1c74b2ea5ce0a65cdabc09e850543bc07f`, Flutter `c750562d26499346e7c43315fba9912951e590d1`, and Business Portal `2ef57f22b1dcae45334dd5f30e896e448073f10b`. Wave A Studio is now verified complete against current code. Wave B magnetic snap is intentionally deferred because current Studio V2 does not mount the historical 2D canvas; do not restore it solely to satisfy historical criteria.

The active P0 is now the remaining cross-client dine-in proof: correlate Flutter → Backend → Business Portal → Admin state for finalize/payment/replay, tips, ambiguous responses and reconnect/background recovery. Backend settlement and Flutter recovery are already component-tested; do not duplicate those paths. Admin-side lifecycle visibility was implemented on 2026-09-14 (backend `GET /api/admin/dine-in/overview` + adminPortal Dashboard panel, both with executable tests, exact-head CI green on backend main `94dfd31` and adminPortal main `730cb18` — this closes the Admin-visibility residual). A single deployed four-surface E2E harness is still unproven. See the CURRENT_STATE.md hygiene notes for route-checker CI wiring, adminPortal dead test suites, Render cold-start and branch-sprawl cleanup.

After the dine-in P0 residual is closed or explicitly bounded, execute the current Studio Wave C rendered evidence (real overflowed scroll, responsive relayout and clipping) before considering any future 2D Studio surface. Continue with financial/control-plane integrity, production operations and adversarial/release testing.

## Non-negotiable engineering loop

**Research → trace producers/consumers → inspect schema/authorization/state machine → implement one coherent batch → test → exact-head CI → diff audit → merge → verify `main` → update Planning → continue.**

Never declare completion from discussion, code existence, or a non-exact CI result. Never duplicate a completed implementation. Close stale/superseded PRs instead of carrying parallel branches.

## Planning rules

- GitHub `main` plus exact CI is implementation evidence; Planning files describe active navigation and residual risk.
- Reconcile PRs, branches, current heads and implementations before starting a new branch.
- Do not weaken tests or type checking to obtain green CI.
- When CI runs, do independent research instead of waiting idle.
- Every substantial merge updates `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`.
