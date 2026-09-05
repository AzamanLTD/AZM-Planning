# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `7afbe1a7318402519aafed6142fdb7364f9df86d` — security dependency remediation, storefront retail catalog alignment, and dine-in recovered-payment notification hardening are merged. The recent backend dine-in exact-head gate passed, and post-merge main verification is green.
- Flutter main: `c13481a37184d602d4555a92306bd7b73a2d8db9` — canonical USDC/GHS FX convergence plus CI checkout modernization and durable dine-in payment recovery; PR #89 merged.
- Business Portal main: `73b9763aa90c258ab496c2c355b78384bbe53568` — Studio Wave A retail preview completion plus Wave B deterministic pointer dragging, magnetic snap, connected-group fusion, and settle animation are merged. PR #85 exact-head CI run #239 passed smoke tests, 46/46 test files / 146 tests, and production build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR verified in this loop.
- Planning main: this reconciliation records the latest verified backend and Studio merges and advances the active P0 frontier.

## Verified recent work

Business Portal PR #85 (`feat(studio): add deterministic pointer drag, snap, and group fusion`) merged at `73b9763a...` after exact-head run #239 passed. It replaces global mouse drag listeners with pointer capture, introduces a measured canvas drag engine, magnetic edge snapping using the shared 6dp threshold/pull sharpness, transitive logical edge-connected group movement, persistence-time grid rounding, and focused contracts. The earlier failed Wave B run was an isolated test assumption and was corrected before the successful exact-head run.

Backend PR #232 (`fix(dine-in): notify business on recovered payment`) merged at `7afbe1a7...`. Durable PAID/CLOSED recovery now also emits the business-owner `DINE_IN_TAB_PAID` notification after closure, preventing a lost-response recovery from leaving Business Portal stale.

Backend PR #230 security remediation remains verified: direct `uuid` 14.0.2, affected nested uuid 11.1.1, esbuild 0.28.2, deepmerge-ts 8.0.2; production-only audit clean and recovery drill passed.

Backend PR #231 retail collection catalog alignment remains verified at `d230aa0d...`, aligning persistent catalog with Flutter and Studio `retail_collection_box` support.

## Active unresolved work

- Cross-client dine-in lifecycle still needs broader executable proof across clients/read models, including finalize/invoice/payment/CLOSED, duplicate/concurrent payment, tips, lost response, reconnect/background, multi-tab races, and Business/Admin convergence.
- Broader canonical invoice caller audit remains queued; current inspected runtime paths use the canonical creation boundary, but exhaustive search evidence was limited by GitHub code-search availability in this environment.
- Studio Wave B is now the verified baseline; future Studio work must build on the deterministic drag engine rather than reintroduce independent layout math.
- Financial/control-plane integrity and production readiness remain queued after the dine-in proof frontier.

## Next P0 sequence

1. Build executable cross-client dine-in lifecycle proof and authoritative convergence coverage.
2. Complete the canonical business-invoice caller/idempotency audit with evidence-driven changes only.
3. Advance Studio only where a concrete remaining interaction gap is demonstrated; then proceed to Wave C device emulation.
4. Continue financial/control-plane integrity across withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
5. Production readiness, concurrency/load/adversarial rehearsal, and release proof.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; customer Flutter converges through API state plus socket signals and durable reread after ambiguous payment outcomes.
- Recovered paid/closed dine-in state must notify the Business Portal after durable reread/closure, not only on the original successful HTTP response.
- POS tax math is business-authoritative: the transactional default `BusinessTaxPreset` is resolved and passed to the canonical invoice tax calculator; authoritative tax lines are recorded with the POS ledger entry.
- Storefront Studio remains a semantic-tree editor with `storefrontStudioRuntimeAdapter` as the customer-facing compatibility boundary. Wave A/B contain no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Studio Wave B uses one immutable measured geometry/token source, pointer capture, continuous preview movement, magnetic edge pull, logical adjacency for connected groups, and integer normalization only when persistence occurs.
- Retail collection is aligned across Flutter widget contract, Studio semantic/runtime adapter, Studio preview renderer, and backend catalog seed.

## Residual risks / hygiene

- GitHub code-search availability is inconsistent for exhaustive caller discovery; use repository-tree/file inspection when verifying remaining invoice callers.
- `StorefrontPhonePreview.jsx` still contains legacy CSS geometry outside the Wave A retail tokenized path; broad tokenization should remain separate measured work.
- Previously identified merged backend branch cleanup remains optional hygiene only if safe GitHub deletion tooling supports it.
- Bundle-size warning (>500 kB chunk) remains a non-blocking production-build warning and should be addressed in a dedicated performance slice rather than mixed into Studio behavior changes.

## Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.