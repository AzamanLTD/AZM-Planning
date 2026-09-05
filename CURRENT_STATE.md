# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `b103c28e6a89dce79feb72b332523828d8065963` — security dependency remediation is merged via PR #230. Exact-head security CI passed production audit, full tests, and database backup/restore; the post-merge main run #896 also passed.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization and durable dine-in payment recovery; PR #89 merged.
- Business Portal main: `563bfecf8187e93a6b5990505ecfee64efd81a34` — Studio Wave A retail preview completion is merged via PR #84 after exact-head Business Portal CI run #234 passed smoke tests, full tests, and build. The Studio preview now has a first-class `retail_collection_box` renderer and 16/16 widget coverage, with measured retail geometry behind the shared token scale.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR verified in this loop.
- Planning main: this reconciliation records the latest verified merges and the next P0 frontier.

## Verified recent work

Business Portal PR #84 (`feat(studio): complete Wave A retail preview coverage`) merged at `563bfecf...` after exact-head CI run #234 passed. It adds measured retail collection tokens, a dedicated preview renderer, registry coverage for `retail_collection_box`, and focused tests for token values, retail props, and 16/16 preview coverage. The renderer uses the one explicit `PREVIEW_SCALE` and mirrors the Flutter retail product shape.

Backend PR #230 (`fix(security): update uuid and tsx dependency lines`) merged at `b103c28e...`. Its final lockfile resolves direct `uuid` to 14.0.2, the affected nested Google/Firebase uuid lines to 11.1.1, esbuild to 0.28.2, and deepmerge-ts to 8.0.2. Exact-head CI passed the production-only npm audit, full test suite, and database backup/restore drill; the post-merge main run also passed.

Backend PR #231 (`feat(storefront): align retail collection widget catalog`) is open and is the current backend follow-on. It seeds `retail_collection_box` into the persistent catalog as a FREE commerce widget with contiguous display order and adds a focused catalog contract test. It must not be promoted to the verified baseline until exact-head CI passes and the merge is verified on main.

## Active unresolved work

- Backend retail collection catalog alignment remains in-flight in PR #231.
- Cross-client dine-in lifecycle still needs executable proof across clients/read models, although the authoritative implementation is hardened.
- Broader canonical invoice caller audit and financial/control-plane integrity remain queued.
- Remaining Studio work moves to Wave B only after the Wave A boundary is kept stable and its measured token contract is reused rather than duplicated.

## Next P0 sequence

1. Finish and verify backend retail collection catalog alignment (PR #231).
2. Advance executable cross-client dine-in proof, including replay/race/tips/reconnect/background and Business/Admin convergence.
3. Complete remaining canonical business-invoice idempotency/replay caller audit.
4. Begin Storefront Studio Wave B pointer-based drag/snap/fusion only after re-validating Wave A contracts.
5. Continue financial/control-plane integrity and production-readiness waves.
6. Concurrency/load/adversarial rehearsal and release proof.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; customer Flutter converges through API state plus socket signals and durable reread after ambiguous payment outcomes.
- POS tax math is business-authoritative: the transactional default `BusinessTaxPreset` is resolved and passed to the canonical invoice tax calculator; authoritative tax lines are recorded with the POS ledger entry.
- Storefront Studio remains a semantic-tree editor with `storefrontStudioRuntimeAdapter` as the customer-facing compatibility boundary. Wave A contains no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Retail collection is now aligned across the Flutter widget contract, Studio semantic/runtime adapter, Studio preview renderer, and backend catalog seed; PR #231 is the final persistence/catalog step still awaiting verification.

## Residual risks / hygiene

- PR #231 is not verified yet; do not treat its open state as a completed catalog change.
- `StorefrontPhonePreview.jsx` still contains independent legacy CSS geometry beyond the new retail tokenized path; broad tokenization beyond Wave A should be handled as a separate measured slice rather than changing multiple widget behaviors at once.
- Previously identified merged backend branch cleanup remains optional hygiene only if the GitHub mutation surface supports safe deletion.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.