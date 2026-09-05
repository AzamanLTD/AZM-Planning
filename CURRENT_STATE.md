# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `b82c995646ee930dfa3b901b012ce3bd07e92f73` — includes OrderTracking atomic initialization/serialization, provider-attempt reference integrity, dine-in item/finalization serialization, canonical invoice idempotency/replay hardening, and POS business-tax authority resolution/persistence after exact-head CI passed. Backend PR #229 is merged at this SHA.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization and durable dine-in payment recovery; PR #89 merged.
- Business Portal main: `eb7b4c004d666f1263987b07219deb58d44a7b44` — API/storefront hardening plus the first Storefront Studio Wave A token-foundation slice; PR #82 merged after exact-head Business Portal CI run #229 passed tests and build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR verified in this loop.
- Planning main: this reconciliation records the current P0 frontier and residual security/release work.

## Verified recent work

Backend POS PR #229 (`fix(pos): use business tax authority for POS totals`) was rebuilt from current backend main and merged only after exact-head CI passed. It resolves the transactional default `BusinessTaxPreset`, delegates tax math to canonical `computeTaxLines`, fails closed on unsupported types, persists authoritative tax lines in ledger metadata, and preserves POS idempotency/replay semantics.

Business Portal PR #82 (`feat(studio): establish measured visual token foundation`) introduced the immutable Studio token contract, measured from the Flutter storefront runtime, with one explicit `PREVIEW_SCALE`; responsive resolution and the runtime adapter now consume the shared token source for their existing defaults. Exact-head Business Portal CI run #229 passed smoke tests, the full test suite, and build before merge.

## Active unresolved work

The two originally prioritized correctness packages — cross-client dine-in lifecycle hardening and POS/invoice tax-authority correctness — are now implementation-complete on their relevant mains. Residual work is proof/release hardening, Studio Wave A completion, dependency security, and broader financial/control-plane coverage.

## Next P0 sequence

1. Complete Storefront Studio Wave A: tokenize the remaining `StorefrontPhonePreview.jsx` renderer geometry and add the missing `retail_collection_box` preview so the Studio covers all 16 runtime widget types.
2. Run the backend production dependency audit and remediate the confirmed npm vulnerabilities on current main, including the known esbuild finding; do not upgrade by guesswork.
3. Advance the cross-client dine-in proof from implementation evidence to executable end-to-end/contract evidence across Flutter → Backend → Business Portal → Admin visibility, including replay/race/reconnect/tips paths.
4. Trace and harden canonical business-invoice creation idempotency/replay callers that remain outside the now-fixed POS boundary.
5. Continue financial/control-plane integrity across withdrawals, wallet, escrow, trades, payroll/EWA, refunds/voids, reservations and admin approvals.
6. Complete tenant/actor/state-machine coverage across remaining Business OS resources.
7. Production readiness: deployment/configuration separation, secrets rotation, migration rollback, observability, worker recovery, anomaly/reconciliation alerts.
8. Concurrency/load/adversarial waves and release rehearsal.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; customer Flutter converges through API state plus socket signals and durable reread after ambiguous payment outcomes.
- POS tax math is now business-authoritative: the transactional default `BusinessTaxPreset` is resolved and passed to the canonical invoice tax calculator; authoritative tax lines are recorded with the POS ledger entry.
- Storefront Studio remains a semantic-tree editor with `storefrontStudioRuntimeAdapter` as the customer-facing compatibility boundary. Studio Wave A must not alter that runtime contract or introduce AI/free-canvas/persistence substitutes.

## Residual risks / hygiene

- Backend production dependency findings still require an actual lockfile/`npm audit --omit=dev` evidence pass and controlled remediation.
- Business Portal Studio Wave A still needs the renderer-wide tokenization and 16/16 widget preview coverage.
- Repository branch hygiene (the previously identified fully merged backend branches) remains cleanup work only if the connected GitHub mutation surface permits safe deletion.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.
