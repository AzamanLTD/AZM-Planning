# AZM Planning — Project Brain

**Status:** ACTIVE / CANONICAL ENGINEERING BRAIN  
**Last reconciled:** 2026-09-03 UTC  
**Purpose:** Make continuation safe and fast across the five AZAMAN repositories.

> ## START HERE
> **Read `START_HERE.md` first. Then read `ROADMAP.md` and `CURRENT_STATE.md`.**
>
> `ROADMAP.md` is the **single active execution plan and priority authority**. `CURRENT_STATE.md` tells you what is true now and what to do next. Do not reconstruct the project from dated session journals.

## Canonical documents

1. **[`START_HERE.md`](START_HERE.md)** — agent entrypoint, read order, immediate queue and non-negotiable workflow.
2. **[`ROADMAP.md`](ROADMAP.md)** — total engineering plan, ordered waves and production definition of done.
3. **[`CURRENT_STATE.md`](CURRENT_STATE.md)** — live cross-repo baseline, verified work, blockers and next exact actions.
4. **[`CONTRACTS.md`](CONTRACTS.md)** — cross-repo producer/consumer contracts, with dine-in and Experience Blueprint as first-class surfaces.
5. **[`FINANCIAL_INVARIANTS.md`](FINANCIAL_INVARIANTS.md)** — money, ledger, escrow, checkout and retry invariants.
6. **[`SECURITY_BOUNDARIES.md`](SECURITY_BOUNDARIES.md)** — actor, permission, tenant and privileged-operation rules.
7. **[`STATE_MACHINES.md`](STATE_MACHINES.md)** — lifecycle transitions and concurrency/state-test rules.
8. **[`ARCHITECTURE.md`](ARCHITECTURE.md)** — living system map and authority boundaries.
9. **[`RELEASE_CHECKLIST.md`](RELEASE_CHECKLIST.md)** — executable production release gate.
10. **[`REPO_GUIDE.md`](REPO_GUIDE.md)** — where to inspect and how to continue safely.

## Historical material

`archive/` contains historical planning policy and is the destination for durable old assessments/session material. Existing dated files outside the canonical set are historical context; they do not override current repository evidence, `ROADMAP.md`, or `CURRENT_STATE.md`.

## Platform backbone

**Identity & Trust → Unified Money/Ledger → Domain Verticals → Experience/SDUI → Realtime/Eventing → Notifications/Social → Observability/Reconciliation**

Backend is the authoritative source for persisted business and financial truth. Flutter, Business Portal and Admin Portal are consumers/control surfaces with explicit authority boundaries; they must converge on committed backend outcomes.

## Engineering loop

**Research → trace producers/consumers → inspect schema/authorization/state machine → implement coherent batch → test → exact-head CI → audit final diff → merge → verify `main` → update Planning → continue.**

Never manufacture completion. Never create duplicate branches for the same logical fix. Never weaken tests/type checking to obtain green CI.
