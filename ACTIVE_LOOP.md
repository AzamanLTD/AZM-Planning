# AZAMAN Active Engineering Loop

**Purpose:** persistent continuation state for long-running engineering work. Resume from here rather than restarting.

## Operating contract

1. Reconcile referenced repos, branches, PRs and SHAs before changes.
2. Research and trace the actual implementation before editing.
3. Execute the whole slice: implement → test → exact-head CI → diff audit → merge → verify main → reconcile Planning.
4. Never duplicate work already on current main.
5. CI failures are defects to diagnose and fix, not reasons to weaken tests.
6. While CI runs, perform independent audits instead of waiting idle.
7. **Studio wave completion requires a criterion-by-criterion acceptance audit against current code and matching executable evidence; a historical green run is insufficient.**
8. Keep every PR <=500 changed lines and include at least one test; split large work before merge.

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Verified current GitHub state — 2026-09-18 UTC

- Backend main: `ab387e77f8de1bb280034dc2075f846d532de00c` (2026-09-18) — §P.4 PR #279 and §P.5-A PR #280 are merged as squashes. P4 remains authoritative across the major money-mutation writer set; P5-A now enforces canonical asset identity at the ledger core, durable conversionIdentity uniqueness, exact conversion provenance hashing and fail-closed account identity. Exact-head CI #1046 succeeded on final P5-A head `fb8fb2a56df0c720ff7b305728bedd801c6120da`; no production financial data changed and live production crypto signing/broadcast remains OFF. Next boundaries are inventory/cost-basis economics, route-aware conversion, GHS liquidity state/reconciliation, Kotani Model A, treasury movement controls, exchange integration/P&L and final PoR/treasury UI work.
- **Open audited finding (2026-09-15, decide before fixing):** `azmStakeService.createStake` never debits or checks `User.azmBalance`, so the Nitro tier gating premium storefront widgets (`PREMIUM_WIDGETS`) is inflatable with unbacked stakes; `completeUnstake` returns nothing because nothing was taken. No doc/spec/test defines stake fund movement — a product-contract gap, not a provable regression. Do not invent debit-on-stake/credit-on-unstake semantics without an owner decision. The adjacent AZM spend surfaces (ad boost, fee discount) share the hardened `_debitAzmWithClient` primitive and showed no new defect.
- Flutter main: `c750562d26499346e7c43315fba9912951e590d1`; PR #90 durable CLOSED-tab recovery is merged, and PR #91 adds the complementary customer payment convergence contract. Exact head `d1dbc94ed890583241a4d338fc2a045cf5cec4a3`, Flutter Quality run `33967426867`, passed Analyze + Test with coverage + upload coverage; merged at `c750562d26499346e7c43315fba9912951e590d1`.
- Business Portal main: `0d8a58d` — Wave C complete (see Studio acceptance gates): #102 `928c19a`, #103 `73cd9ce`, #104 squash `0d8a58d`, exact-head + main-head CI green on each. Earlier history: PR #88 pointer palette insertion, PR #89/#91/#92/#94/#95 Wave A token/wiring slices, PR #93 bounded device-frame scroll, PR #96 Wave A completion, and PR #98 dine-in lifecycle invalidation proof are merged and verified. PR #96 exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf` passed smoke/tests/build; PR #98 exact head `85560129a19470e40a70175a8049eb3cffde8655`, Business Portal CI run `33975428983`, passed smoke/tests/build.
- Admin Portal main: `60faed7` (2026-09-15) — the dine-in lifecycle Dashboard projection (`730cb18`) plus PR #98 (merge `60faed7`): the four vitest lib suites now run in CI conventionally (vitest 4.1.11 devDependency, `npm test` = `vitest run src/lib`, CI `npm run test`), replacing PR #97's ad-hoc undeclared `npx vitest` step. Exact-head run #251 green at `7c07a3c`; main-head run #252 green at `60faed7`. The "no dedicated dine-in projection" finding and the vitest CI hygiene finding are both closed.

### Studio acceptance gates

- **Wave A — VERIFIED / complete:** current main consumes grounded shared preview tokens across all 16 widget renderers plus action/fallback/selection/chrome geometry and the device frame. PR #96 adds an executable completion guard for zero inline numeric CSS `px` literals, renderer registry coverage, and frame tokenization. Keep this wave closed unless current Flutter measurements change.
- **Wave B — REOPENED / partial — 2D prerequisite decision made:** current V2 uses Pointer Events for palette insertion with capture, pointermove/up/cancel, before/after hit testing and click suppression. `StorefrontStudioV2.jsx` mounts the semantic layer tree + phone-preview stage, not the legacy `StorefrontCanvas.jsx` magnetic-snap surface. The legacy canvas still contains a coherent shared-token drag/snap implementation, but mounting it would create a second layout interaction model before V2 has explicit coordinate/persistence authority. **Do not reintroduce the historical 2D canvas just to satisfy old criteria.** A future 2D surface must first define coordinate/persistence authority and an executable geometry contract; only then should snap/fuse/settle be ported.
- **Wave C — COMPLETE (2026-09-15):** the JSDOM-rendered slice is MERGED (PR #100, squash `b9b5481`). The browser-level slice: #102 (the fix) MERGED at squash `928c19a`; #103 (scroll half, 471 lines) MERGED at squash `73cd9ce`; #104 (responsive/token half) was rebuilt from the post-#103 main as head `6885980` (162/0, two files, one commit, mergeable clean), exact-head workflow_dispatch CI green at `6885980` (smoke, vitest, Playwright Chromium E2E, build), then squash-merged by Sugru as `0d8a58d` with the exact head SHA pinned; post-merge main-head push CI green at `0d8a58d`. Wave C is closed with genuine end-to-end rendered evidence at both the DOM and browser level. Original #101 remains closed/superseded at 824 lines against the <=500 contract.
- Backend PR #233 is merged and verified; no duplicate backend payment path or replacement proof is needed.
- Flutter PR #91 is merged at `c750562d26499346e7c43315fba9912951e590d1`. Its test locks the payment POST → authoritative tab reread → durable CLOSED recovery boundary and preserves the original failure when durable proof is absent.
- Business Portal PR #98 is merged at `7140658f4d66cada3d6c3155c085638105fe484e`; exact head `85560129a19470e40a70175a8049eb3cffde8655`, run `33975428983`, proves all supported DINE_IN_TAB_* lifecycle notifications invalidate canonical dine-in roots while unrelated order events do not.
- **P0 status: VERIFIED at cross-client contract scope.** Backend financial/replay authority, Flutter durable recovery and Business Portal owner-notification/query convergence each have current executable evidence. Socket payloads remain convergence signals only.
- **Admin-side dine-in lifecycle visibility (2026-09-14, VERIFIED):** backend endpoint + admin Dashboard projection implemented with executable tests on both sides; exact-head CI green on backend `94dfd31` and adminPortal `730cb18`.
- **Residual integration gap:** no single deployed four-surface E2E harness currently correlates live Backend, Flutter, Business Portal and Admin state through one finalize/payment/replay/reconnect scenario. The Admin projection now gives that harness an Admin surface to read.

### Priority after current reconciliation

1. Exercise and, where justified, strengthen current Studio Wave C rendered evidence rather than reopening Wave B's deferred magnetic snap without a 2D surface.
2. DONE 2026-09-15 — the adminPortal vitest suites are wired into CI (PR #98, merge `60faed7`; runs #251/#252 green). The remaining dine-in residual is the deployed four-surface E2E harness; build it only when a testable deployed environment makes it valuable. Do not duplicate existing component proofs.
3. **DONE 2026-09-18 — §P.4 authoritative liability ledger:** PR #279 merged as `f3f72a65`; full 223-suite / 1,593-test battery and exact CI/recovery evidence verified. Do not reopen P4 except for newly demonstrated defects.
4. **DONE 2026-09-18 — §P.5-A multi-asset ledger accounting identity:** PR #280 squash-merged as `ab387e77`; final head `fb8fb2a`; 224-suite / 1,613-test battery green, exact-head CI #1046 green, 499 changed lines, and real-PostgreSQL concurrency/provenance proofs verified. Do not reopen P5-A except for newly demonstrated defects.
5. **DONE 2026-09-18 — §P.5-B inventory lot authority:** PR #281 squash-merged as `39a4977`; final head `fb56d06`; 225-suite / 1,628-test battery green, exact-head CI #1048 green (24.1 min, single slow runner, zero flakes), 469 changed lines, real-PostgreSQL conservation/concurrency/idempotency proofs verified. InventoryLot/InventoryLotConsumption are the ONLY inventory authority; CorporatePurchaseLog stays an untouched audit log; no route settlement, GHS liquidity, spread or P&L in this slice. Do not reopen P5-B except for newly demonstrated defects.
6. Advance the next financial architecture slice: route-aware quote authority wiring, then GHS liquidity states/reconciliation, Kotani Model A, treasury controls, exchange abstraction, P5-E cost-basis realization and PoR/treasury read models.
5. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence. Dine-in proof should be marked verified at the narrowest scope actually executed; do not promote component/contract proofs to a cross-client deployed-E2E claim.
