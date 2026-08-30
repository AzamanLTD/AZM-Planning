# Pyrax's Planning Plan

**Repository:** `AzamanLTD/AZM-Planning`  
**Status:** IN PROGRESS  
**Last updated:** 2026-08-30 (UTC)

## Purpose

This repository is the central project brain for AZM. It records architecture, flows, implementation state, decisions, risks, verification and roadmap across the actual product repositories.

It is deliberately separate from application source code.

## Files

- `README.md` — index and master architecture.
- `Pyrax's frontend plan.md` — customer application.
- `Pyrax's backend plan.md` — authoritative API/domain/money platform.
- `Pyrax's businessPortal plan.md` — merchant operating system.
- `Pyrax's adminPortal plan.md` — platform operations/control.
- This file — maintenance protocol for the planning brain.

## Required content model

Each repository plan should maintain these sections as relevant:

1. Identity/purpose
2. Master architecture relationship
3. Repository-specific architecture
4. Major user/business flows
5. State/data authority
6. Security/trust boundaries
7. Financial implications
8. Realtime/eventing
9. Social implications
10. Performance
11. Implemented work
12. Verified work
13. Active work
14. Planned work
15. Deferred/rejected work
16. Known risks
17. Cross-repository dependencies
18. Verification strategy
19. Chronological change history
20. Agent continuation instructions

## Update protocol

When substantial work occurs:

1. Read the relevant plan before editing code.
2. Inspect the actual repository state and surrounding callers/contracts.
3. Make the coherent implementation batch.
4. Verify with the appropriate tests/checks when the batch is ready.
5. Update the plan with the actual result.
6. Update `README.md` if master architecture/status changed.
7. Record the UTC date/time.
8. State whether the work is PLANNED, IN PROGRESS, IMPLEMENTED, VERIFIED, DEFERRED or REJECTED.
9. Record important discoveries even if they do not result in code changes.
10. Do not rewrite history to make the document look cleaner.

## What counts as a substantial update

Update the plan for:

- new architecture or domain boundaries;
- state-machine changes;
- financial/payment/escrow changes;
- authorization/security changes;
- data-model/schema changes;
- new shared platform primitives;
- cross-repository contracts;
- major UI/UX flows;
- realtime/eventing changes;
- performance architecture;
- significant bug/race-condition fixes;
- completed verification milestones;
- discovered risks that change priorities.

Tiny refactors that do not alter behavior need not receive individual entries, but their surrounding batch should be recorded if they materially affect the architecture.

## Status rules

**PLANNED:** intended but not started.  
**IN PROGRESS:** implementation or investigation is active.  
**IMPLEMENTED:** code/config exists, but the required verification has not necessarily completed.  
**VERIFIED:** implementation has passed the appropriate tests/review or other explicit evidence.  
**DEFERRED:** intentionally postponed with reason recorded.  
**REJECTED:** intentionally not being pursued with rationale recorded.

Never use “complete” as a substitute for evidence.

## Cross-repository rule

If a change crosses a contract boundary, update every affected plan. Examples:

- backend API ↔ Flutter service/model
- storefront contract ↔ Portal editor ↔ customer renderer
- payment/escrow ↔ order state
- domain event ↔ notification/realtime/social feed
- admin action ↔ backend authorization/audit

The backend remains authoritative for financial and protected state.

## Batch/CI rule

Do not trigger expensive CI for every micro-change. Accumulate a meaningful, internally coherent batch, review it deeply, then run the expensive verification gate. If verification fails, record the failure and correction in the affected plan.

## Research rule

Before modifying an unfamiliar area, inspect the existing implementation, callers, data model, API contract, tests, deployment behavior and adjacent repositories where applicable. Prefer extending an existing primitive over creating a parallel one.

## Continuation rule for any agent

An agent entering the project should be able to work from these documents without needing prior chat history. If something important is missing from the plan, fixing the plan is itself part of the work.

When uncertain, preserve existing behavior until the surrounding contract is understood. For financial, authorization, inventory and state-machine changes, prefer fail-closed behavior and explicit transitions.
