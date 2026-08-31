# AZAMAN — Current Engineering State — 2026-08-31

## Purpose

This file is a durable continuation point for future engineering sessions. It records verified repository state, the current architectural direction, and work that must not be duplicated.

## Repository baseline

Repositories in scope:

- `AZM-backend` — authoritative identity, money, domain state, APIs and events.
- `AZM-frontend` — Flutter consumer experience.
- `AZM-businessPortal` — merchant operating/control surface.
- `AZM-adminPortal` — administrative/control-plane surface.
- `AZM-Planning` — architecture, roadmap and verification memory.

## Verified PR state

As of 2026-08-31 there are no open pull requests under the AzamanLTD organization from the current repository search.

Important recent Backend history:

- PR #52 `fix: converge concurrent escrow satisfaction responses` is merged.
- PR #53 `fix: make admin force release atomic from disputed state` is closed and unmerged.
- PR #54 `fix: make admin force release an atomic financial claim` is closed and unmerged.

PRs #53/#54 were experiments around repository-write/tooling limitations. They must not be treated as implemented product work. Do not resurrect either branch as-is.

## Backend CI baseline

Backend `main` is at the post-PR #52 baseline. The normal `Azaman Test Suite` returned green after the temporary force-release workflow experiments were removed.

The failed/cancelled runs around PR #53/#54 were not evidence that the established Backend CI was intrinsically broken. They were contaminated by temporary workflow/branch manipulation. Future work must use the existing CI as the validation gate rather than introducing self-modifying patch workflows.

## Architectural rule: one authoritative financial mutation path

For financial operations:

1. Controller validates request and authorization.
2. Canonical domain service owns the financial transition.
3. The authoritative state transition is claimed atomically inside the transaction before money moves.
4. Balance, fee, ledger/history and durable financial state changes occur in the same transaction where applicable.
5. Realtime/socket events are convergence signals emitted only after authoritative commit.
6. Clients refetch canonical HTTP state instead of treating socket payloads as a second financial database.
7. Retries and concurrent callers converge to the committed canonical state where the operation is safely idempotent.

Never introduce a second ledger, second socket transport, second event bus, or second domain state machine to solve a convergence problem.

## Current Admin force-release finding

Current Backend `adminController.forceRelease` still performs a separate conditional `DISPUTED -> PAID` mutation and then calls `p2p.completeTrade`, whose canonical financial claim only accepts `PAID -> COMPLETED`.

This leaves an avoidable committed intermediate state between an Admin decision and the financial settlement engine.

The intended correction is NOT to create another settlement implementation. The canonical `p2p.completeTrade` engine should own the privileged Admin `DISPUTED -> COMPLETED` claim, with an explicit authorization input from the Admin controller. The same transaction must perform the claim before balances/fees/history/profit are changed.

Required regression evidence:

- two concurrent Admin force-release calls against one `DISPUTED` trade;
- exactly one financial completion path;
- exactly one payout/fee/history/profit result;
- no `DISPUTED -> PAID` intermediate state committed by the controller;
- a failed settlement must not leave the trade stranded in `PAID`;
- ordinary `PAID -> COMPLETED` callers must remain unchanged;
- Admin audit/realtime behavior must remain post-commit and non-duplicative.

This work is researched but **not implemented** in `main`.

## Realtime convergence already completed

Recent merged batches establish the following pattern:

### Escrow
Backend escrow satisfaction/release convergence was hardened in PR #52. Concurrent opposite-party satisfaction now converges to the committed `SETTLED` state rather than producing a false failure, while the single financial release claim remains authoritative.

### Business Portal
The existing singleton realtime/query bridge invalidates canonical React Query projections. It now covers order/escrow, business notifications, invoice payment and invoice void convergence. It does not patch financial state directly.

### Flutter
The existing singleton Socket.IO service is the realtime transport. Recent merged work routes invoice/order/balance events into canonical provider/API refresh paths rather than creating local financial truth.

### Admin Portal
The existing privileged Admin socket and realtime hook invalidate canonical financial/control-plane queries. The portal does not treat socket payloads as authoritative balances.

## Cross-repo contract rule

For every new backend event or state transition, research all four consumers before changing the producer:

`Backend producer → Admin consumer → Business consumer → Flutter consumer`

Then verify:

- event name;
- room/audience;
- payload identity fields;
- post-commit timing;
- idempotency/replay behavior;
- consumer query/provider invalidation;
- listener registration/removal lifecycle;
- stale-read protection;
- duplicate listener risk.

## CI rules

Do not modify CI merely to make a dashboard green.

Before implementation:

- inspect the current workflow;
- inspect the latest green run;
- inspect any recent failed/cancelled run;
- determine whether the cause is source, test, environment, workflow, or branch/concurrency;
- prefer deterministic dependency installation and existing repository gates;
- never introduce a temporary self-modifying patch workflow to bypass repository tooling limitations.

A failed run becomes an engineering investigation, not something to hide or blindly retry.

## Current highest-value work queue

### P0 — Financial truth

1. Complete the Admin force-release atomicity correction described above.
2. Verify Admin audit + realtime convergence for the corrected path.
3. Continue the financial mutation audit across P2P, escrow, withdrawal, refund and direct payment paths.
4. Mature reconciliation from transaction-row inspection into explicit invariant/exception detection.
5. Establish/propagate correlation IDs across financial operations.

### P0 — Cross-repo retail convergence

1. Continue auditing canonical order lifecycle across Backend, Business Portal and Flutter.
2. Verify duplicate-tap/retry-after-timeout/stock-race behavior.
3. Ensure delivery/refund/settlement events have one producer and one convergence path per client.

### P0/P1 — Administrative control plane

1. Finish the Admin dispute/financial governance lifecycle around the canonical Backend mutation paths.
2. Burn down pre-existing Admin type-system debt without weakening type checks.
3. Continue reconciliation exception ownership and realtime convergence.

### P0/P1 — Business Portal

1. Keep the existing production-build CI gate operational.
2. Prove realtime query convergence against every authoritative business-order/invoice/escrow event.
3. Avoid introducing a second state store or socket layer.

## Duplicate-work protection

Before starting a task, search:

- current `main` implementation;
- merged PRs for the same feature;
- closed/superseded PRs;
- existing event producers/consumers;
- existing services/helpers;
- current Planning documents;
- all four application repositories.

If a previous branch contains useful work but is stale, port the intent to a fresh branch based directly on current `main`; never blindly revive a stale branch.

## Definition of done

A substantial batch is complete only after:

1. affected architecture and callers were researched;
2. affected schema/migrations were checked;
3. authorization was checked;
4. idempotency/concurrency was checked;
5. realtime/event consumers were checked;
6. regression coverage exists for the failure mode;
7. exact CI head is green;
8. the final diff has been audited for duplication;
9. merged `main` is re-read after merge;
10. this Planning state is updated with evidence.
