# AZAMAN Engineering Brain — START HERE

**Status:** ACTIVE — PRIMARY CONTINUATION ENTRYPOINT  
**Last verified:** 2026-09-03 UTC  
**Repositories:** `AZM-backend`, `AZM-adminPortal`, `AZM-businessPortal`, `AZM-frontend`, `AZM-Planning`

> Any agent continuing AZAMAN engineering MUST start here, then read `ROADMAP.md`, `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`. Do not restart from historical session journals.

## Mission

Bring AZAMAN to production-ready correctness by establishing who owns truth, who may mutate it, how state transitions, how concurrency behaves, and how every client converges after failure/realtime loss.

## Current priority

The backend current main is `301a5795898f4b7de3b69c8156afe027f82e5155`, containing the verified kiosk hardening from PR #144 and POS atomicity/inventory hardening from PR #145. Planning PR #27 is stale and closed.

The current implementation loop is PR #146: `fix/pos-recipe-duplicate-line-consumption`. It fixes under-consumption when the same restaurant product appears on multiple POS lines by aggregating product quantities before recipe requirements are calculated. Exact-head CI must pass before merge.

Immediately after #146, continue the first unchecked P0s: transaction-time POS catalog authority, location/table/customer boundary validation, idempotency payload binding, canonical tax/invoice authority, then dine-in `confirmAndPay` across all clients and Admin visibility. Continue with `updateAccruedWages()` producer/consumer tracing, Admin financial mutation coverage, tenant/state/realtime matrices, production operations and red-team/load testing.

## Non-negotiable engineering loop

**Research → trace producers/consumers → inspect schema/authorization/state machine → implement one coherent batch → test → exact-head CI → diff audit → merge → verify `main` → update Planning → continue.**

Never declare completion from discussion, code existence, or a non-exact CI result. Never duplicate a completed implementation. Close stale/superseded PRs instead of carrying parallel branches.

## Planning rules

- GitHub `main` plus exact CI is implementation evidence; Planning files describe active navigation and residual risk.
- Reconcile PRs, branches, current heads and implementations before starting a new branch.
- Do not weaken tests or type checking to obtain green CI.
- When CI runs, do independent research instead of waiting idle.
- Every substantial merge updates `CURRENT_STATE.md`, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json`.
