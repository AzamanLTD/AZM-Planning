# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `b82c995646ee930dfa3b901b012ce3bd07e92f73` — includes OrderTracking atomic initialization/serialization, provider-attempt reference integrity, dine-in item/finalization serialization, canonical invoice idempotency/replay hardening, and POS business-tax authority resolution/persistence after exact-head CI passed. Backend PR #229 is merged at this SHA.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization and durable dine-in payment recovery; PR #89 merged.
- Business Portal main: `84bf2c78630c6c194c95292079b205eb469cd0ac` — Studio token foundation plus first-class retail collection semantic/runtime plumbing; PR #82 and #83 merged after exact-head Business Portal CI passed.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR verified in this loop.
- Planning main: this reconciliation records the current P0 frontier and dependency audit evidence.

## Verified recent work

Business Portal PR #83 (`feat(studio): model retail collection as a first-class runtime widget`) was merged at `84bf2c78...` after exact-head Business Portal CI run #231 passed. The Studio semantic model now recognizes legacy `retail_collection_box` and the runtime adapter preserves that canonical widget type. The actual visual renderer is still the remaining Wave A gap.

Backend PR #230 is open for security remediation. Its exact-head CI run #885 produced the first real production audit after upgrading direct `uuid`/`tsx`: esbuild's high finding disappeared; production audit now reports 9 vulnerabilities (6 moderate, 3 high). Remaining high is `deepmerge-ts` through Prisma config, and remaining uuid findings are transitive through Firebase Google-cloud storage dependencies. This branch is not mergeable until lockfile remediation, full tests, recovery drill, and final clean CI are verified.

## Active unresolved work

- Storefront Studio Wave A visual completion remains: tokenize `StorefrontPhonePreview.jsx` geometry/type scale from actual Flutter sources and provide the `retail_collection_box` preview renderer so all 16 runtime types are represented.
- Backend production dependency remediation remains active after the first audit exposed exact transitive paths.
- Cross-client dine-in lifecycle still needs executable proof across clients/read models, although the authoritative implementation is hardened.
- Broader canonical invoice caller audit and financial/control-plane integrity remain queued.

## Next P0 sequence

1. Finish Storefront Studio Wave A visual renderer/tokenization and prove 16/16 widget coverage.
2. Remediate backend production dependency advisories with narrow, evidence-driven overrides/upgrades and exact-head CI.
3. Advance executable cross-client dine-in proof, including replay/race/tips/reconnect/background and Business/Admin convergence.
4. Complete remaining canonical business-invoice idempotency/replay caller audit.
5. Continue financial/control-plane integrity and production-readiness waves.
6. Concurrency/load/adversarial rehearsal and release proof.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; customer Flutter converges through API state plus socket signals and durable reread after ambiguous payment outcomes.
- POS tax math is business-authoritative: the transactional default `BusinessTaxPreset` is resolved and passed to the canonical invoice tax calculator; authoritative tax lines are recorded with the POS ledger entry.
- Storefront Studio remains a semantic-tree editor with `storefrontStudioRuntimeAdapter` as the customer-facing compatibility boundary. No AI generation, free-canvas geometry model, or localStorage persistence is permitted in Wave A.

## Residual risks / hygiene

- PR #230 currently has known transitive production audit findings that require remediation; do not merge based on the direct dependency updates alone.
- `StorefrontPhonePreview.jsx` still contains independent CSS geometry and lacks `retail_collection_box` in `WIDGET_RENDERERS`.
- Previously identified merged backend branch cleanup remains optional hygiene only if the GitHub mutation surface supports safe deletion.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.
