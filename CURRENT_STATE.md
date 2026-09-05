# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `d230aa0d5e865f3b39b828f2101a01afb20ac762` — security dependency remediation and storefront retail catalog alignment are merged. PR #230 passed production audit/full tests/recovery, and PR #231 passed the same exact-head gate before merge.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization and durable dine-in payment recovery; PR #89 merged.
- Business Portal main: `563bfecf8187e93a6b5990505ecfee64efd81a34` — Studio Wave A retail preview completion is merged via PR #84 after exact-head Business Portal CI run #234 passed smoke tests, full tests, and build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR verified in this loop.
- Planning main: this reconciliation records the retail catalog merge and advances the active P0.

## Verified recent work

Business Portal PR #84 (`feat(studio): complete Wave A retail preview coverage`) merged at `563bfecf...` after exact-head CI run #234 passed. It adds measured retail collection tokens, a dedicated preview renderer, registry coverage for `retail_collection_box`, and focused tests for token values, retail props, and 16/16 preview coverage.

Backend PR #230 (`fix(security): update uuid and tsx dependency lines`) merged at `b103c28e...`; exact-head CI and post-merge main CI were green. The production dependency tree is clean: direct `uuid` 14.0.2, affected nested uuid 11.1.1, esbuild 0.28.2, deepmerge-ts 8.0.2.

Backend PR #231 (`feat(storefront): align retail collection widget`) merged at `d230aa0d...` after exact-head CI passed production audit, full tests, and database recovery. The persistent catalog now includes the same `retail_collection_box` contract already supported by Flutter and Studio, as a FREE commerce widget with contiguous display order 0–15.

## Active unresolved work

- PR #232 (`fix(dine-in): notify business on recovered payment`) is now the active backend slice. It addresses a concrete stale-read-model hole in ambiguous payment recovery: durable PAID/CLOSED recovery previously returned without the normal `DINE_IN_TAB_PAID` business notification.
- Cross-client dine-in lifecycle still needs broader executable proof across clients/read models, including reconnect/background/race/tip scenarios.
- Broader canonical invoice caller audit and financial/control-plane integrity remain queued.
- Studio Wave B remains queued until the next high-value dine-in proof slice is verified.

## Next P0 sequence

1. Verify/merge PR #232 after exact-head backend CI and post-merge verification.
2. Continue executable cross-client dine-in proof, emphasizing payment replay/races/tips/reconnect/background and authoritative Business/Admin convergence.
3. Complete remaining canonical business-invoice idempotency/replay caller audit.
4. Begin Storefront Studio Wave B pointer-based drag/snap/fusion after re-validating Wave A contracts.
5. Continue financial/control-plane integrity and production-readiness waves.
6. Concurrency/load/adversarial rehearsal and release proof.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; customer Flutter converges through API state plus socket signals and durable reread after ambiguous payment outcomes.
- POS tax math is business-authoritative: the transactional default `BusinessTaxPreset` is resolved and passed to the canonical invoice tax calculator; authoritative tax lines are recorded with the POS ledger entry.
- Storefront Studio remains a semantic-tree editor with `storefrontStudioRuntimeAdapter` as the customer-facing compatibility boundary. Wave A contains no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Retail collection is aligned across Flutter widget contract, Studio semantic/runtime adapter, Studio preview renderer, and backend catalog seed.

## Residual risks / hygiene

- PR #232 is not verified yet; do not treat its open state as completed.
- `StorefrontPhonePreview.jsx` still contains legacy CSS geometry beyond the Wave A retail tokenized path; broad tokenization should remain a separate measured slice.
- Previously identified merged backend branch cleanup remains optional hygiene only if safe GitHub deletion tooling supports it.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.