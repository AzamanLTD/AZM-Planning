# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `526f659f83af5d7fc708a0de964abad707d5a6a7` — security dependency remediation, storefront retail catalog alignment, recovered dine-in payment notification hardening, and executable invoice-to-closure orchestration proof are merged. The latest backend exact-head gate and database recovery drill are green.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — canonical USDC/GHS FX convergence, CI checkout modernization, durable dine-in payment recovery, and focused recovery parsing proof are merged.
- Business Portal main: `73b9763aa90c258ab496c2c355b78384bbe53568` — Studio Wave A retail preview completion plus Wave B deterministic pointer dragging, magnetic snap, connected-group fusion, and settle animation are merged. PR #85 exact-head CI run #239 passed smoke tests, 46/46 test files / 146 tests, and production build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening; no open PR verified in this loop.
- Planning main: this reconciliation records the verified backend lifecycle proof and Flutter recovery proof before the next Studio gate.

## Verified recent work

Business Portal PR #85 (`feat(studio): add deterministic pointer drag, snap, and group fusion`) merged at `73b9763a...` after exact-head run #239 passed. It replaces global mouse drag listeners with pointer capture, introduces a measured canvas drag engine, magnetic edge snapping using the shared 6dp threshold/pull sharpness, transitive logical edge-connected group movement, persistence-time grid rounding, and focused contracts.

Backend PR #232 (`fix(dine-in): notify business on recovered payment`) merged at `7afbe1a7...`. Durable PAID/CLOSED recovery now also emits the business-owner `DINE_IN_TAB_PAID` notification after closure, preventing a lost-response recovery from leaving Business Portal stale.

Backend PR #233 (`test(dine-in): prove invoice-to-closure orchestration`) merged at `526f659f...`. It proves FINALIZED → invoice → PAID → CLOSED with tip, deterministic `DINE_IN_TAB:<tabId>` identity, atomic closure, customer settlement signal, and paid replay without a second charge.

Flutter PR #90 (`test(dine-in): prove durable client payment recovery`) merged at `bf058958...`. Durable reread is accepted only when the requested tab is authoritatively CLOSED; malformed, wrong-tab, and non-CLOSED responses are rejected.

Backend PR #230 security remediation remains verified: direct `uuid` 14.0.2, affected nested uuid 11.1.1, esbuild 0.28.2, deepmerge-ts 8.0.2; production-only audit clean and recovery drill passed.

Backend PR #231 retail collection catalog alignment remains verified at `d230aa0d...`, aligning persistent catalog with Flutter and Studio `retail_collection_box` support.

## Active unresolved work

- Cross-client dine-in lifecycle still needs broader executable proof across clients/read models, including finalize/invoice/payment/CLOSED, duplicate/concurrent payment, tips, lost response, reconnect/background, multi-tab races, and Business/Admin convergence. Core backend orchestration and Flutter recovery proof are now in place.
- Broader canonical invoice caller audit remains queued; current inspected runtime paths use the canonical creation boundary, but exhaustive search evidence was limited by GitHub code-search availability in this environment.
- Studio Wave B is the verified baseline; a Wave C device-emulation PR is now under exact CI gate. Do not mark it complete until its exact head is green and merged.
- Financial/control-plane integrity and production readiness remain queued after the dine-in proof frontier.

## Next P0 sequence

1. Finish the cross-client dine-in executable proof package and authoritative Business/Admin convergence coverage.
2. Complete the canonical business-invoice caller/idempotency audit with evidence-driven changes only.
3. Verify/merge Studio Wave C device emulation, then address remaining concrete responsive/layout gaps.
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
