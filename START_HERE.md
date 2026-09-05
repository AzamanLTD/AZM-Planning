# AZAMAN Engineering Brain — START HERE

**Status:** ACTIVE — PRIMARY CONTINUATION ENTRYPOINT  
**Last verified:** 2026-09-05 UTC  
**Repositories:** `AZM-backend`, `AZM-adminPortal`, `AZM-businessPortal`, `AZM-frontend`, `AZM-Planning`

> Any agent continuing AZAMAN engineering MUST start here, then read `ROADMAP.md`, `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`. Do not restart from historical session journals.

## Mission

Bring AZAMAN to production-ready correctness by establishing who owns truth, who may mutate it, how state transitions, how concurrency behaves, and how every client converges after failure/realtime loss.

## Current priority

Current verified main baselines are Backend `ad6110213f5a859fd9e47db75d0f36682c32974e`, Flutter `c750562d26499346e7c43315fba9912951e590d1`, and Business Portal `2ef57f22b1dcae45334dd5f30e896e448073f10b`. Wave A Studio is now verified complete against current code. Wave B magnetic snap is intentionally deferred because current Studio V2 does not mount the historical 2D canvas; do not restore it solely to satisfy historical criteria.

The active P0 is now the remaining cross-client dine-in proof: correlate Flutter → Backend → Business Portal → Admin state for finalize/payment/replay, tips, ambiguous responses and reconnect/background recovery. Backend settlement and Flutter recovery are already component-tested; do not duplicate those paths. Admin-side lifecycle visibility and a single deployed four-surface E2E harness are still unproven.

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
