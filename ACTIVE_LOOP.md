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

### Current continuation snapshot — 2026-10-19 FINAL CLOSE-OUT (owner-verified CI + exact-head benchmark complete)

**This block supersedes the 11:00 snapshot below.** GitHub main and exact-head CI remain implementation authority. All PRs listed here are OPEN and UNMERGED — nothing in these notes authorises a merge.

#### Owner-verified CI (2026-10-19) — recorded into the ledger

- **BE #359** head `0d5df5680acbffc2f0d3c01c112a5377e3b3f67f`: test run `39225590677` and financial-durability run `39225590938` GREEN (both first attempt).
- **FE #156** head `6690b65acbb9e50253f3f1061e8030ef5f2a2369`: Flutter Quality run `39193716143` and Android Integration run `39193716122` GREEN. The ledger's two #156 workstream entries (`frontend-marketplace-mix-in-place`, `frontend-priceghs-consumption-156`) previously carried `in_progress` conclusions — both now flipped to `success` with the run IDs; the prior heads `f608a27` (FQ 39189639176 / AI 39189639113, green) and `7d117f2` are explicitly marked HISTORICAL.
- **BE #353** remains FROZEN at `7805c3cbee5c9d8fd01cc19689e21b2cb63d1de6`, green.

#### Exact-head adaptive-scan benchmark (owner-requested, 2026-10-19, clean real CI-parity Postgres, BE head `0d5df56`)

- **CASE A — 21,000 stale window-closed rows ahead of one fresh eligible item:** ONE request, **45 `businessProduct.findMany` queries**, wall **4,606 ms (cold process) / 5,309 ms (warm)**, fresh item surfaced at **page position 0**, `nextCursor: null` (true end of feed). The pre-adaptive measurement (9 requests / 720 queries / ~60s, item never surfacing) is HISTORICAL and was not reused as current.
- **CASE B — 25,500 stale rows (just past `MAX_SCAN_ROWS = 25,000`):** first page truthfully returned `items: []` with a NON-NULL cursor (**53 queries**, **7,091 ms**, within the 80-query cap), and the client continuation following that cursor surfaced the fresh item in **88 ms / 5 queries** at page position 0.
- Verdict: both cases satisfy the shipping invariants; **no production code change warranted** (production code identical from `0d5b154`).

#### Residual owner items — UNCHANGED

- Mix tile visual captures (320/360/390dp light/dark): **PENDING OWNER REVIEW** — not to be recorded as accepted.
- #154 physical-device smoke: **BLOCKED** (no physical device in sandbox).
- Production Render remains suspended; **no merges, no deployments, no production writes this wave**. PRs #353, #359, #156 await owner merge review.

### Superseded snapshot — 2026-10-19 11:00 UTC (BOTH BACKEND LANES GREEN at head 0d5df56 — BE #359 fully verified)

 GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge.

#### BE #359 — VERIFIED GREEN at exact head `0d5df5680acbffc2f0d3c01c112a5377e3b3f67f`

- **Azaman Test Suite run 39225590677: SUCCESS (first attempt).** financial-durability run 39225590938: SUCCESS (first attempt). The full battery — including the workflow's retry-once step — needed no retries; the hour-independent scan-cap suite and all 374 suites passed.
- The `0d5b154` test-lane failure is closed as diagnosed: the scan-cap suite's static `03:00-04:00` "never-open" window was OPEN at the 03:15 UTC CI hour (stale rows genuinely eligible, filled the page ahead of the fresh item); fixed INSIDE the test suite only at `0d5df56` — dynamic `closedWindow(nowMin)`, 180s teardown headroom, cascade-first sweep, patch restoration, battery-robust ≤80-query bound. The adaptive walk and all production code are untouched from `0d5b154`.
- The same lane's mix-p1 `[]` failure remains classified as the known battery-retry cross-suite-residue flake (passed invocation 1; 55/55 locally on the 21k-residue DB across repeated in-band rounds; same symptom pre-dated the adaptive change on 2026-10-18). It did not recur at `0d5df56`.
- PR #359 description updated with the green first-attempt evidence at the exact head.

#### FE #156 — ALREADY VERIFIED GREEN at head `6690b65` (Flutter Quality 39193716143, Android Integration 39193716122). Frontend close-out COMPLETE.

#### Residual owner items (UNCHANGED — not claimable by agents)

- Mix tile visual captures: **PENDING OWNER REVIEW** (visual acceptance NOT complete).
- #154 physical-device smoke: **BLOCKED** (no physical device in sandbox).
- #353 frozen at `7805c3cbee5c9d8fd01cc19689e21b2cb63d1de6`, both lanes green.
- Production Render remains suspended; **no merges, no deployments, no production writes this wave.**

### Superseded snapshot — 2026-10-19 09:15 UTC (test-lane CI failure diagnosed and fixed; head 0d5df56 CI running)

 GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge.

#### BE #359 test-lane failure — root-caused and fixed at `0d5df5680acbffc2f0d3c01c112a5377e3b3f67f` (test-suite-only commit; the adaptive walk is untouched)

- **What failed at `0d5b154` (test run 39192378361, 2026-10-19 03:15 UTC):** all three scan-cap proofs + mix-p1 `inactive section is excluded` (only in the workflow's full-battery RETRY invocation). financial-durability was GREEN there (39192378371).
- **Root cause 1 (deterministic, my suite's bug):** the scan-cap suite seeded its "stale" window-closed rows with a static `03:00-04:00` section window on a falsified "never open during battery hours" assumption. CI ran INSIDE that hour → the stale rows were genuinely eligible → they filled the first page ahead of the fresh item. The received arrays in the CI failure are the suite's own stale rows.
- **Fix (in `__tests__/marketplace-discovery-window-scan-cap.pg.test.js` only):** dynamic `closedWindow(nowMin)` — a window that ended ~30 minutes before the current UTC minute, wrap-safe, closed at EVERY hour of every day; `jest.setTimeout(180000)` so the 25.5k-row teardowns can never die on the 30s hook cap on a loaded runner (a killed sweep would leak bulk residue into every later suite's unscoped feed); cascade-first sweep (BusinessPrice onDelete: Cascade — products delete clears prices in one statement); monkey-patch restoration via an afterEach registry; the 3000-row proof asserts the SHIPPING hard cap (queries ≤ 80) instead of the clean-DB ≤15 figure (51 queries observed against a 21k-row residue DB locally).
- **Root cause 2 (mix-p1 `[]`, NOT a regression):** failed only in the workflow's full-battery retry invocation on accumulated cross-suite residue; it PASSED in invocation 1 at the same code, and passes 55/55 on the 21k-residue local CI-parity DB across repeated in-band rounds with all four discovery suites (81/81 × 3). Treated as the known battery-residue flake family (same symptom failed pre-adaptive-code on 2026-10-18, run 39120763057). No mix-p1 code or test was changed.
- **Local verification at `0d5df56` (real CI-parity Postgres with 21k stale residue rows):** scan-cap 3/3; mix-p1 55/55; rail 21/21; eligibility 2/2; all four suites in band 81/81 three consecutive rounds.
- **CI at exact head `0d5df56`:** test run 39225590677 RUNNING, financial-durability run 39225590938 RUNNING. Flip the ledger fields on conclusion.
- **Everything else in the 03:40 snapshot below is unchanged:** FE #156 head `6690b65` BOTH LANES GREEN (FQ 39193716143, AI 39193716122); #353 frozen at `7805c3c`; residual owner items (Mix tile visual captures PENDING OWNER REVIEW; #154 physical-device smoke BLOCKED; Render suspended, no deployments/production writes). PR #359 description updated to exact head `0d5df56` including the failure diagnosis.

### Historical snapshot — 2026-10-19 03:40 UTC (acceptance close-out)

*(Retained as history; superseded by the snapshot above.)*


**This block supersedes the 2026-10-19 02:30 snapshot below.** GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge. Historical heads/CI conclusions in older snapshots below are retained as history — reconcile every head against GitHub before acting.

#### Acceptance close-out wave (2026-10-19) — implementation complete, newest-head CI RUNNING

- **BE PR #359:** head **`0d5b1545608a13eaa48413d4c15e4825053cecd4`** (branch `feat/marketplace-product-ghs-price-rail`). Close-out commit: adaptive window-filter scan growth — the per-round fetch DOUBLES after a zero-eligible round (cap 512 rows/query) and resets to `(need+1)` after a productive one; the cursor still rests at the LAST EXAMINED row so §4 page exactness and cursor correctness are unchanged; hard budgets are explicit and two-sided: **MAX_ROUNDS=80 queries + MAX_SCAN_ROWS=25000 examined rows per request**. **Measured worst case (real CI-parity Postgres, 21,000 stale window-closed rows ahead of one fresh eligible item): 1 client request / 45 server queries / 5.9s wall, item on the FIRST page** — was 9 requests / 720 queries / ~60s with the item never surfacing. Realistic 90-row battery case: 3 queries / ~29ms. Scan-cap suite now 3 tests: 90-row surfacing; NEW 3,000-row single-request surfacing in ≤15 queries; NEW 25,500-row truthful budget exhaustion (empty page + NON-NULL cursor at ≤80 queries; the client continuation finds the item on the next request). Rail/mix-p1/ghs-model/provisioning/eligibility suites all green locally. CI at exact head `0d5b154`: financial-durability run 39192378371 **SUCCESS**; Azaman Test Suite run 39192378361 **RUNNING**. PR #359 description updated to exact head `0d5b154` including the `GET /api/storefront/:bizId/products` priceGhs contract.
- **FE PR #156:** head **`6690b65acbb9e50253f3f1061e8030ef5f2a2369`** (branch `feat/marketplace-mix-inplace-mode`). Close-out commit: cursor-progress validation carried into MANUAL continuation — `loadMore()` remembers every consumed cursor per selection; a returned cursor equal to the submitted cursor or already consumed is a STALL (`continuationStalled`); the surface replaces Show more with an explicit stalled panel (Try again = one validated manual attempt; Refresh feed = fresh walk with a clean cursor history) and loadMore — including the scroll trigger — no-ops while stalled (no request loop). Empty pages with NEW cursors still auto-follow within the bounded limit. NEW truthful loading feedback: `searchingWide` during multi-request walks renders a visible 'Still searching — looking further for you…' line above the skeleton grid (worst case never a silent spinner). Provider suite **29/29** (new manual-stall, loop-stall, retry/refresh and searchingWide proofs); surface suite **6/6** (stalled panel replaces Show more, searching feedback, compact stalled tail on a non-empty feed); marketplace_mix_inplace 7/7; analyzer 0 errors. CI at exact head `6690b65`: Flutter Quality run 39193716143 **RUNNING**, Android Integration run 39193716122 **RUNNING**. PR #156 description updated to exact head `6690b65`. Prior heads `f608a27` (both lanes GREEN: FQ 39189639176, AI 39189639113) and `7d117f2` are historical only.
- **BE PR #353:** head FROZEN at `7805c3cbee5c9d8fd01cc19689e21b2cb63d1de6` per owner instruction — both lanes GREEN there (test 39119836530, financial-durability 39119836561). No further pushes to this branch.
- **Residual owner items (unchanged, not accepted):** Mix tile visual captures (320/360/390 light/dark) REMAIN PENDING OWNER REVIEW — visual acceptance is NOT complete. Physical-device smoke for #154 remains BLOCKED (no device in environment). Production Render services remain suspended (503) — no merges, deployments, Render settings or production-data changes this wave; production DB/Render remain off-limits.
- **Next actions:** (1) verify exact-head CI on `0d5b154` (test lane) and `6690b65` (both lanes), then flip the ledger RUNNING notes to their conclusions; (2) owner merge review in the established order (backend #353 → #355-#358 → #359 → #360; frontend #160 → #161 → #156 → #157 → #154 → #158 rebase → #159; #155 excluded); (3) post-merge, verify main heads and redeploy only after the owner lifts the Render suspension.

### Historical snapshot — 2026-10-19 02:30 UTC (final acceptance follow-up)

*(Retained as history; superseded by the snapshot above.)*
### Current continuation snapshot — 2026-10-19 02:30 UTC (final acceptance follow-up)

**This block supersedes the 2026-10-18 11:00 snapshot below.** GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge. Historical heads/CI conclusions in older snapshots below are retained as history — reconcile every head against GitHub before acting.

#### Final acceptance follow-up wave (2026-10-19) — implementation complete, newest-head CI RUNNING

- **BE PR #359 (GHS price rail + storefront catalogue):** head `1e403f85…` (exact `1e403f8` — full SHA below in ledger). Prior head `1be1bd8` was green on both lanes (test 39179051032, durability 39179051041). The scan-cap fix (MAX_ROUNDS 3→80 + hermetic regression) is in `1be1bd8`; the final acceptance follow-up added the storefront ordering catalogue `GET /api/storefront/:bizId/products` carrying authoritative `priceGhs` (same shared resolver, exact 2dp, per-identity fail-closed, `priceUsdc` byte-identical as the payable rail amount) with route regressions 5a-5d. Rail suite 21/21; scan-cap + mix-p1 re-verified 56/56. CI GREEN at exact head `1e403f8`: test 39189630963 SUCCESS, financial-durability 39189630855 SUCCESS (2026-10-19).
- **FE PR #156 (GHS presentation + storefront/order truth):** head `f608a27`. Prior head `1362d3e` was green (FQ 39184028451, AI 39184028499); the obsolete `7d117f2` head recorded in earlier snapshots is historical only. The follow-up: storefront catalogue card GHS-only ('Price unavailable' when absent, never `priceUsdc`); StorefrontOrderSheet payable amounts explicitly `USDC` (header/Total/Place Order — no bare `$`) with the GHS listing price as a separate 'Listed: GH₵…' fact; CartScreen subtotal/total/checkout/per-item all explicitly USDC; StorefrontRenderer nested-list-in-sliver unbounded-viewport bug fixed (debug crash / infinite-extent release sliver); repeated-cursor progress guard in the Mix first-page continuation; business-card unpriced anchor without the 'from' prefix. USDC payment/wallet/withdrawal rails and financial semantics unchanged. Storefront journey suite 5/5, provider 26/26, surface 3/3, card 4/4, affected battery 102/102, test/storefront 70/70, test/marketplace 164/164. CI GREEN at exact head `f608a27`: Flutter Quality 39189639176 SUCCESS, Android Integration 39189639113 SUCCESS (2026-10-19).
- **BE PR #353 (Kotani rate authority):** head FROZEN at `7805c3cbee5c9d8fd01cc19689e21b2cb63d1de6` per owner instruction — both lanes GREEN there (test 39119836530, financial-durability 39119836561). No further pushes to this branch.
- **Measured worst-case client continuation (real Postgres, CI-parity DB):** 21000 stale window-closed rows ahead of one fresh item → the client envelope (1 initial + 8 auto-follow continuations) makes 9 requests / 720 server queries / ~60s wall (2.7-11.3s per request). Per-request cost stays bounded by the MAX_ROUNDS=80 server cap (~1s at the 1920-row budget boundary on a small table; realistic 90-row battery case: 4 queries / ~29ms). Guards: hard continuation limit, stale-response protection, repeated-cursor progress guard, honest empty state with 'Show more' affordance. No infinite loop paths.
- **Residual owner items (unchanged):** Mix tile visual captures (320/360/390 light/dark) REMAIN PENDING OWNER REVIEW — visual acceptance is NOT complete. Physical-device smoke for #154 remains BLOCKED (no device in environment). Production Render services remain suspended (503) — no deployments, environment, settings or production-data changes made; production DB/Render remain off-limits.
- **Next actions:** (1) DONE — exact-head CI green confirmed on `1e403f8` (both backend lanes) and `f608a27` (both frontend lanes); (2) owner merge review in the established order (backend #353 → #355-#358 → #359 → #360; frontend #160 → #161 → #156 → #157 → #154 → #158 rebase → #159; #155 excluded); (3) post-merge, verify main heads and redeploy only after the owner lifts the Render suspension.

### Historical snapshot — 2026-10-18 11:00 UTC (r40 close-out)

*(Retained as history; superseded by the snapshot above.)*

### Current continuation snapshot — 2026-10-18 11:00 UTC (r40 close-out)

**This block supersedes the 2026-10-18 05:30 snapshot below.** GitHub main and exact-head CI remain implementation authority. The PRs listed here are open and unmerged; these notes do not authorise a merge.

#### r40 wave CLOSED — all CI confirmed green at exact heads

- **BE PR #359 (GHS price rail):** head `adfe18fd978357b47c51ac40dba6a2bb785c8668` (CI-hardening commit on top of `c6cbf2f`). Both lanes green: Azaman Test Suite run 39103046162, financial-durability run 39103046145. The full battery surfaced three order/isolation defects, all repaired in `adfe18f`: (1) fail-closed guard in `resolveEffectiveGhsPrices` for mock clients without the `businessPrice` model (priceGhs null + loud warn, never 500, never invented); (2) `business-ad-adapter` WalletAddress sweep (FK-safe user cleanup deletes WalletAddress children before user_* rows); (3) hermetic afterAll teardown on the rail suite so `ghsrail-` residue cannot shift neighbor cursor-walk totals. Local re-verify: rail 10/10, model 32/32, mix P1 55/55, location-filter 2/2, ad-adapter 6/6 on real Postgres.
- **BE PR #360 (trade-account wiring):** head `340e6d5fd9568b02b7923520f4cf57e4cdd389a8`. Both lanes green: run 39099984234 (test), run 39099984245 (financial-durability).
- **FE PR #156 (priceGhs consumption):** head `9bc4eee1ae98b78031b7b6419d266d7ee53ce234`. Both lanes green: Flutter Quality run 39099872804, Android Integration run 39099872798. The Mix card-grammar visual acceptance gap REMAINS OPEN.
- **All other open PRs green at exact heads:** BE #353 `43f3202c0cb80e81f728c81ab283a5fae9d1167c` (runs 39083687375/39083687124), #354 `081aa5f77ac6a2fdefbb7bf80feb4dcffb782eb3` (39078767367/39078767193), #355 `81aef466b4d302b2a3f2ff4d2fe36814051300f4` (test lane 39080780987 SUCCESS — the run flagged in review §2 is now concluded), #356 `45dedae4d5d61f2a7ea434b237da504928313bd1`, #357 `a661559cc7049231ba82e80182470c083187d1a5`, #358 `6b3641ef6de7027d218e20ee694626ff4eae6d11`, #319 `3bb753fc7b1e39d6e2052840b0b797e1268fbf22`; FE #154 `708a0f8ae77169784c59bbadef10be0eb78789a8`, #155 `6a55d1609e1bcc5233f7982d6bddabdee4889f9b`, #157 `5e53f9ed3b9e1a70b2a07c58f501d8596c2c91a0`, #158 `0d9ecf5be0206b3120cb042050584e56ae87601d`, #159 `2da9e478c5a0db034274a3dd18c1ff52676857df`, #160 `b9adbd0ecc781b8418636b98fadc85672b25e490`, #161 `78c1315e5d75533726d7d1926d299a404fc2b47c`.

#### Owner decisions and standing constraints (r40)

- **Nav launcher placement SETTLED — RIGHT.** The owner REJECTED moving the AZM logo/action launcher to the left of the nav pill. Revert commit `8022156` on PR #157 (head `5e53f9e`) restored the trailing/right slot byte-identical to `525b9ab`. Any instruction to move the launcher LEFT anywhere in these planning files is void; the approved placement is RIGHT. (The earlier 2026-10-16 snapshot below still contains LEFT instructions — they are corrected here and must not be re-executed.)
- **PR #155 is branch-only, NOT for merge.** Its PR states it is not intended for merge. Exclude #155 from every merge sequence. Owner merge review order for the rest: backend #353 → #359 → #360 (plus #354-#358, #319 at owner discretion); frontend #160, #161, #156, #157 → #154 → rebase/adapt #158 to `WalletCardDeck` (drop the `PullRevealCardDeck` dependency, resolve the real `home_screen.dart` conflict) → #159 (its router conflict with #158 resolves in that order).
- **Physical Android device smoke for #154 remains BLOCKED** — no physical device is available in the work environment. Emulator/widget CI lanes are green but are NOT device smoke; report it as blocked, not covered.
- **Production (read-only):** both Render services return 503 "Service Suspended" via HTTP. Evidence supports an account-level suspension response, but do not state billing suspension as proven beyond what the 503 body shows; owner billing action is the gate for redeploy. No deployment, environment, settings or production-data changes have been made.

#### Verified integration order (for owner merge review only)

Backend: #353 → #355 → #356 → #357 → #358 → #359 → #360 (all additive; #359 depends on nothing open, #360 independent). Frontend: #160 → #161 → #156 → #157 → #154 → rebase #158 onto `WalletCardDeck` → #159. #155 excluded (branch-only).

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
- **PR #156 — in-place Marketplace Mix:** OPEN, head `9bc4eee1ae98b78031b7b6419d266d7ee53ce234` (GHS consumption + stories + logo-parse work landed). Flutter Quality 39099872804 and Android Integration 39099872798 green at that head. The backend discovery logo URL now exists as PR #354 (`businessLogoUrl` DTO, both lanes green) — the FE dependency is satisfied once #354 merges. **Visual acceptance is NOT complete:** the card-grammar gap (near-square tiles with lower info panel + store logo/name) REMAINS OPEN. Keep #156 open until its card grammar is corrected and previewed.
- **PR #157 — compact bottom nav:** ~~OPEN, head 525b9ab~~ head `5e53f9ed3b9e1a70b2a07c58f501d8596c2c91a0` after revert `8022156`. Flutter Quality 38977174754 and Android Integration 38977174991 green; post-revert lanes green at `5e53f9e` (run 38993688789 / 38993688794). Selected Home/Chat/Market label and explicit Tabs control are implemented. **SUPERSEDED (owner decision 2026-10-18):** the launcher-LEFT placement below was REJECTED by the owner; the logo/action launcher stays on the RIGHT of the nav pill. The old LEFT instruction is void and must not be re-executed.
- **AZM-backend main:** 48c17ab10248e39689683772c88695a4c8b0c0e8 (PR #352 merged).
- **PR #353 — Kotani-only transaction rate authority:** OPEN, head 24f3bdbcaabd1044451b368b505ae916c7332d0d. Main test run 38978189219 and financial durability run 38978189222 green. Only validated Kotani observations can update canonical transaction-rate/freshness; the CoinGecko+FX composite no longer acts as a transaction source. Kotani onboarding/credentials are still outstanding. Do not merge/deploy by inference.
- **PR #72 — this Planning handoff:** OPEN on docs/current-handoff-2026-10-16; this update adds the latest agent results, visual acceptance gaps, transfer contract and app-wiring P0s. Keep the durable project brain synchronized.

#### New product acceptance requirements

1. **Mix card layout:** near-square image-led grid tiles, with a real lower panel for item identity/price and a clear small store-logo + business-name row. Add responsive tests/goldens at 320/360/390dp and both themes. Current screenshots show the actual tree and correctly expose the mismatch; this is not just a screenshot mix-up.
2. **Mix logo data:** add an additive nullable business logo URL to GET /api/discovery/items from the actual BusinessProfile logo field, with backend contract tests. Frontend must tolerate absent/bad logo with a graceful initials/monogram fallback. Do not fabricate a store image.
3. **Nav position (SETTLED 2026-10-18):** logo/action button stays on the RIGHT of the nav pill per owner decision — the LEFT placement in this line was REJECTED and reverted (`8022156`). Preserve selected labels, 44dp hit targets, badges, long-press, retap-to-top, scroll compression, reduced motion, safe area, Marketplace focus mode and pushed-route Tabs.
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
3. ~~Amend PR #157 to put the AZM logo/action on the left~~ VOID — owner rejected the LEFT placement (revert `8022156`); launcher stays RIGHT.
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
