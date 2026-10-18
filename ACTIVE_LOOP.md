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
8. Keep each slice coherent, testable, and reviewable. There is no arbitrary changed-line target; split work when scope is logically too large, not to satisfy a line-count metric.

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Current continuation snapshot — 2026-10-18 05:30 UTC

**This block supersedes the 2026-10-16 23:30 snapshot below.** GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge.

#### Rate-truth + GHS-presentation follow-up wave (2026-10-17/18)

- **PR #353 — Kotani-only transaction rate authority (§1 follow-up COMPLETE):** head `43f3202`. Provenance gate on `getFreshServerRateGhsPerUsdc` (non-KOTANI_PAY label → RATE_UNAVAILABLE; manual mutation newer than observation → RATE_UNTRUSTED); legacy `liveUsdToGhs` USD/GHS fallback removed from canonical paths; provider-owned canonical fields rejected from the admin settings whitelist with a loud 400; production arming requires https + verified host allowlist (`KOTANI_VERIFIED_PRODUCTION_HOSTS`, ships EMPTY — Kotani onboarding must confirm the production hostname in writing before the operator adds it). Local battery 343/343 on disposable DB; both CI lanes green on `43f3202`.
- **PR #359 — Marketplace product GHS price rail (§3 backend):** OPEN, head `c6cbf2f` on branch `feat/marketplace-product-ghs-price-rail`. Additive `priceGhs` on `GET /api/business/:bizId/products` + my-products + product detail: the exact 2dp GHS string of the product's eligible current BusinessPrice row, resolved through the SAME `resolveEffectiveGhsPrices` authority as the Mix feed. Never a USDC→GHS conversion; absent/unconfirmed/MIGRATION/future-dated/archived/BRANCH rows → `priceGhs: null` per-identity fail-closed. 10/10 real-PostgreSQL proofs; neighbors green in isolation (mix-p1 55/55, ghs-price-model 32/32, provisioning 3/3). CI in progress at snapshot time — do not treat as green until confirmed.
- **PR #156 — Mix FE consumption of priceGhs (§3 frontend):** head `9bc4eee`. `BusinessProduct.priceGhs` strict-parsed (exact-2dp string only; numbers/1dp/3dp/blank → null, never derived from priceUsdc). ProductCard, BusinessCard cheapest anchor, ServiceExperienceStage (3 sites), flip book, native menu journey and vertical retail stage PREFER server-confirmed GHS; truthful USDC label stays otherwise. Cart mutation rail unchanged (USDC escrow). 7/7 new proofs; neighbor suites green (vertical stage 6/6, routes 2/2, store query 4/4, service stage 4/4, flip book 6/6, mix models 37/37). CI in progress at snapshot time. Note: the Mix card-grammar visual acceptance gap from the 2026-10-16 snapshot REMAINS OPEN and is unaffected by this additive consumption work.
- **PR #360 — Trade-account rail wiring (§4 false-affordance fix):** OPEN, head `340e6d5`. `tradeAccountController.getSupportedMethods` and `deleteTradeAccount` existed but were never registered, so the FE's `GET /api/trade-accounts/supported-methods` and `DELETE /api/trade-accounts/:id` 404'd. Two additive route registrations (`/supported-methods` above the `/:id` catch-all); controller bodies untouched (ownership-guarded soft archive, never hard delete). 5 structural proofs; r274 tranche2 25/25 on real Postgres. CI in progress at snapshot time.
- **§4 audit result (read-only elsewhere):** every other suspect FE rail checked out against backend main — contacts sync/recent/invite, AZM loyalty (history/summary/rates/leaderboard), AZM spend family, dine-in tabs, susu groups/vouches, liability-contract, proof-of-residency, close-friends stories (GET/POST/DELETE all mounted), trade-account list/add/approved/upload-screenshot. The two orphaned trade-account routes in #360 were the only real gaps.
- **Production (read-only check 2026-10-18):** both Render services still return 503 "Service Suspended" (account-level suspension; owner billing action required). No deployment or production-data work has been performed. Production DB/Render settings remain strictly off-limits.

#### Standing verification order for this wave

1. Confirm exact-head CI green on #359 (test + financial-durability), #156 (Flutter Quality + Android Integration), #360 (test + financial-durability). CI in progress ≠ green.
2. Owner merge review in dependency order: #353 → #359 → #360 (backend), #156 after its Mix card-grammar visual fix.
3. Post-merge: verify main heads and redeploy only after Render suspension is lifted by the owner.




### Current continuation snapshot — 2026-10-16 23:30 UTC

**This block supersedes the earlier snapshot below.** GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge.

#### Verified code/PR status

- **AZM-frontend main:** ee12a90fdf254d201984f52eca93bb39b9486912.
- **PR #154 — wallet pouch/deck:** OPEN, head 708a0f8ae77169784c59bbadef10be0eb78789a8. Flutter Quality run 38971032050 and Android Integration run 38971032048 green. Toggle-to-tuck, visible selector label "Visa", stable typewriter footprint and spacing pass landed; viewports 320×667, 360×740, 390×844 and 412×915 verified in widget evidence. Physical Android device smoke remains outstanding. The new inter-card transfer button/page is a separate feature, not part of #154.
- **PR #155 — settings redesign:** OPEN, head 6a55d1609e1bcc5233f7982d6bddabdee4889f9b. Flutter Quality run 38969900926, Android Integration run 38969900788, green. Deposit now routes to DepositScreen, Edit Profile to the real editing flow, cache action clears the in-memory image cache honestly, bogus Language picker was removed, biometric state wording is honest, and FAQ claims were corrected/tested. Full app localization is deferred; authoritative 2FA status is absent until the backend exposes it. This PR is ready for independent review, not auto-merge.
- **PR #156 — in-place Marketplace Mix:** OPEN, head 267dd8bcab2770e294a182dd47cefc637c575e59. Flutter Quality 38975394627 and Android Integration 38975394586 green. In-place mode, All/unfiltered default, dial-to-discovery category mapping, die animation, and taxonomy bubbles are implemented. **Visual acceptance is NOT complete:** mix_offer_card.dart still uses a horizontal row with a 96×96 thumbnail on the left and text to the right, visually similar to store/list cards. The approved Mix direction is a near-square tile grid with product image above, product name/price in a distinct lower information panel, and store logo + store name in that lower panel. The discovery DTO/model currently carry businessName but not a logo URL; backend DTO work is required. Keep #156 open until its card grammar is corrected and previewed.
- **PR #157 — compact bottom nav:** OPEN, head 525b9ab0bbd1d1c3999e56fb90ce9068e2a903b0. Flutter Quality 38977174754 and Android Integration 38977174991 green. Selected Home/Chat/Market label and explicit Tabs control are implemented. **Position is not yet the founder's requested layout:** the brand launcher still uses the trailing slot (to the right) and the compact group isn't explicitly centered. Desired: AZM logo/action to the LEFT of the compact pill, close to it, with the combined control group visually centered and responsive. Keep #157 open for this placement correction.
- **AZM-backend main:** 48c17ab10248e39689683772c88695a4c8b0c0e8 (PR #352 merged).
- **PR #353 — Kotani-only transaction rate authority:** OPEN, head 24f3bdbcaabd1044451b368b505ae916c7332d0d. Main test run 38978189219 and financial durability run 38978189222 green. Only validated Kotani observations can update canonical transaction-rate/freshness; the CoinGecko+FX composite no longer acts as a transaction source. Kotani onboarding/credentials are still outstanding. Do not merge/deploy by inference.
- **PR #72 — this Planning handoff:** OPEN on docs/current-handoff-2026-10-16; this update adds the latest agent results, visual acceptance gaps, transfer contract and app-wiring P0s. Keep the durable project brain synchronized.

#### New product acceptance requirements

1. **Mix card layout:** near-square image-led grid tiles, with a real lower panel for item identity/price and a clear small store-logo + business-name row. Add responsive tests/goldens at 320/360/390dp and both themes. Current screenshots show the actual tree and correctly expose the mismatch; this is not just a screenshot mix-up.
2. **Mix logo data:** add an additive nullable business logo URL to GET /api/discovery/items from the actual BusinessProfile logo field, with backend contract tests. Frontend must tolerate absent/bad logo with a graceful initials/monogram fallback. Do not fabricate a store image.
3. **Nav position:** logo/action button at left, adjacent to the nav pill, combined cluster centered. Preserve selected labels, 44dp hit targets, badges, long-press, retap-to-top, scroll compression, reduced motion, safe area, Marketplace focus mode and pushed-route Tabs.
4. **Balance↔Visa transfer:** separate feature. Add a circular action anchored to the stable overlap seam between the two Home cards; open a full-height pull-up surface matching Receive. Reuse the EXACT same card component/model/provider so customisation on Home and transfer stays in sync. Bottom tabs "To Visa" and "From Visa" must be both tappable and swipeable. The page shows both cards, direction-of-money flow lines/particles, and reuses the existing AmountKeypad. On amount submission, require a review/confirmation stage and authoritative operation status; maintain durable idempotency, exact asset/currency semantics, and secure auth.
   - **Financial blocker:** azm_visa_card.dart is explicitly a SAMPLE and the current DemoCardProgramme only supplies demo display data and session-only PIN operations—no issuance, funding or reverse-transfer authority. No transfer may show false success or mutate wallet balances locally. Keep execution disabled / honestly labelled until the real Flutterwave card programme confirms Ghana availability, card denomination, funding and reverse-transfer APIs, and backend ledger/settlement authority exists. Provider discovery is an active prerequisite, not permission to invent endpoints.
5. **App-wide wiring audit:** report returned against frontend ee12a90 and backend 48c17ab. Start with the reported P0s below; audit was read-only and no fixes were made. Re-check exact current source before implementing any finding.

#### App wiring audit — highest-priority reported defects

- **P0: Send Money:** send_money_screen.dart posts /finance/internal-transfer, which the audit did not find in the backend route registry. The UI searches arbitrary AZM ID/username, whereas the existing /friends/transfer/send rail is friendship-ID scoped. Product direction is direct AZM-ID sending; preserve that intended experience by designing a real backend-authoritative, retry-safe transfer contract or document a deliberate product change. Never fake a friendship or route to vendor pool transfer.
- **P0: web demo fallback:** Splash's apiClient.get('/health') resolves under /api/health, while the verified health route is at root /health. A healthy backend may be mistaken for an outage and web may silently enable seeded demo mode. Use the root health URL and test healthy/unhealthy cases.
- **P0: Worker EWA:** worker_ewa_screen.dart prepends /api despite ApiClient already adding it, yielding a double prefix and a broken real EWA request.
- **P0: auction bidding:** auction target selection reads /p2p/my-ads, reported as demo-only/no backend route. The canonical mine endpoint is /ads/mine; fix using the existing service/DTO and add a selection + live-path widget test.
- **P1 double-prefix family:** worker_provider.dart, worker swaps/time off, message actions, forward dialog, call history and notification preferences contain calls requiring root/path normalization review; /api/groups has no matching route and should map to the actual group-chat contract rather than only stripping a prefix.
- **P1 marketplace stories:** /stories/business/:bizId is reported as demo-only and expects groups, while live backend /marketplace/business/:id/stories returns stories; needs a response adapter and tests. Coordinate with Mix because both touch Marketplace Home (use a distinct method/file or sequence branches).
- **P1 dead affordances:** no-op notification snooze and chat Document action; storefront promo CTA is a no-op when configured. Implement supported outcomes or remove/disable the promise. Dead unused methods can be a later cleanup after call-site proof.
- Audit evidence report is advisory pending lead-level source verification; do not label all app controls verified from a path scan alone.

#### Current execution order and parallel ownership

1. Finish independent review of PR #155; no merge until owner instruction.
2. Amend PR #156 for the near-square store-branded item card, and have a separate backend agent add the nullable logo field/tests; front end must gracefully render before backend deploy.
3. Amend PR #157 to put the AZM logo/action on the left and center the combined nav cluster.
4. Keep PR #154's verified interaction/layout fixes; complete Android device smoke. Specify Balance↔Visa transfer separately and do not implement a false live-money action.
5. Parallel P0 remediations: web-health/demo-mode guard; EWA API path; auction ad-source route; cross-repo AZM-ID Send Money authority. Separate owners/file sets; the Send Money lane requires a canonical financial transaction design and adversarial/idempotency tests.
6. Then P1 API-path family, story response adapter, and no-op affordance fixes. Wiring audit stays read-only.
7. PR #353 rate-authority fix, Firebase registered-device push test, paid Redis/distributed-mode gate and Discovery GHS price gate continue in parallel.

#### Cross-chat continuity contract

On every new chat: read START_HERE.md, this latest current continuation block, the current dated snapshot at the top of CURRENT_STATE.md, and EXECUTION_LEDGER.json; reconcile GitHub heads, exact CI and Render. Treat memory as a convenience, not the implementation authority. Do not duplicate verified work or infer approval/merge/deploy from agent completion.

### Verified current GitHub state — 2026-09-19 UTC

- Backend main: `baaa4e2359d404fe2bc0b81638c2314b279cf976` (2026-09-19) — §P.4, §P.5-A, §P.5-B, §P.5-C and §P.5-D are merged. P5-C route-aware quote authority is closed at squash `19671d7`; P5-D evidence-backed GHS liquidity authority is closed at squash `baaa4e2` from exact head `22ebebd`. P5-D has 55 dedicated real-PostgreSQL proofs and a 227-suite / 1,719-test clean battery; exact-head CI #1060 succeeded. P5-D preserves 271C external-rate freshness, uses exact-pesewa GHS accounting, keeps `SystemFiatPool` as compatibility projection, follows recorded withdrawal regime, and sends the originally reserved GHS amount despite later rate drift. Main is at the verified squash merge SHA. The available workflow lookup does not independently expose a post-merge push-triggered run for the squash commit, so none is inferred. Next boundary: §P.5-E Model B settlement / cost-basis realization.
- **Open audited finding (2026-09-15, decide before fixing):** `azmStakeService.createStake` never debits or checks `User.azmBalance`, so the Nitro tier gating premium storefront widgets (`PREMIUM_WIDGETS`) is inflatable with unbacked stakes; `completeUnstake` returns nothing because nothing was taken. No doc/spec/test defines stake fund movement — a product-contract gap, not a provable regression. Do not invent debit-on-stake/credit-on-unstake semantics without an owner decision. The adjacent AZM spend surfaces (ad boost, fee discount) share the hardened `_debitAzmWithClient` primitive and showed no new defect.
- Flutter main: `0fb8d24791088490df0ef5b1088e44cfe557c5a9` (2026-10-01) — PR #130 retail checkout recovery step 1 squash-merged from exact head `9421ffb9e5df0a1cbfa945cd73d32534d13f6352`; exact-head Flutter Quality and Android Integration both green; independent review found no blocker. Post-merge push CI is not inferred because the available workflow lookup exposes PR-triggered runs only.
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
4. **DONE 2026-09-18 — §P.5-A multi-asset ledger accounting identity:** PR #280 merged as `ab387e77`; final head `fb8fb2a`; 224-suite / 1,613-test battery green and exact-head CI #1046 green. Do not reopen P5-A except for newly demonstrated defects.
5. **DONE 2026-09-18 — §P.5-B inventory lot authority:** PR #281 squash-merged as `39a4977`; final head `fb56d06`; exact-head CI #1048 and post-merge main CI #1049 both succeeded. 225 suites / 1,628 tests, route-check, Prisma validation, production audit and database recovery verified. Do not reopen P5-B except for newly demonstrated defects.
6. **DONE 2026-09-19 — §P.5-C route-aware quote authority:** PR #284 squash-merged as `19671d7`; exact-head CI #1055 and post-merge main CI #1056 succeeded. Do not reopen P5-C except for a demonstrated regression.
7. **DONE 2026-09-19 — §P.5-D evidence-backed GHS liquidity authority:** PR #285 squash-merged as `baaa4e2`; exact head `22ebebd`; exact-head CI #1060 succeeded; 227 suites / 1,719 tests, 55 dedicated real-PostgreSQL proofs, route-check/prisma/audit/recovery clean. Do not reopen P5-D except for a demonstrated regression.
8. **NEXT — §P.5-E Model B settlement / cost-basis realization:** connect evidence-backed GHS receipt/liquidity, inventory lots, customer USDC liability, actual acquisition cost and realized customer spread without quote-only P&L or synthetic GHS/inventory.
9. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Frontend economic-operation lifecycle — active slice (2026-10-01)

- Step 1 is **MERGED/VERIFIED** at frontend main merge `0fb8d247...` from PR #130 exact head `9421ffb...`.
- **NEXT: deep-dive step 2 — failure classification.** Audit the LIVE production retail checkout error taxonomy from transport through `ApiClient`, `StorefrontService`, and the UI, distinguishing definitive pre-economic failures, ambiguous/unknown outcomes, authentication refresh, domain conflicts, malformed successful responses, and user-correctable validation failures.
- Step 3 remains a required cross-repository contract proof: verify frontend durable fingerprint and backend `checkoutFingerprint` cover exactly the same immutable economic intent before declaring the shared contract complete.
- Preserve the no-second-identity rule: do not create another UUID/idempotency helper, registry, or operation lifetime model.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence. Dine-in proof should be marked verified at the narrowest scope actually executed; do not promote component/contract proofs to a cross-client deployed-E2E claim.
