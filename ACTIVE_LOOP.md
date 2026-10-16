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


### Current continuation snapshot — 2026-10-16 UTC

**Read this block first.** Historical dated snapshots below are still useful evidence, but their old "next" bullets are not the current task queue unless explicitly re-confirmed.

#### Exact current code/PR status

- **AZM-frontend main:** `ee12a90fdf254d201984f52eca93bb39b9486912`.
- **PR #154 — wallet pouch/card deck:** OPEN and UNMERGED at `8d084462823cab43a34959b0b1ceb1c494ecdb2c`. Exact-head Flutter Quality run #566 (`38935295368`) and Android Integration run #274 (`38935295369`) passed. The state-aware z-order, per-card semantics, tucked-page scrolling, and stale-PIN guard are implemented. The actual Android hardware smoke test remains outstanding. Current founder follow-ups not yet implemented on this head: tapping the active Balance selector again should tuck the card; selector text should read "Visa" (card branding may stay "AZM Visa"); compact/stabilise Home's top vertical rhythm without shrinking touch targets below 44dp or breaking the intentional Recent Activity scroll/handoff.
- **PR #155 — settings drawer + Settings IA:** OPEN and UNMERGED at `cc1da24f6bb9ed2e84784784fa4e05101c6b98bf`. Exact-head Flutter Quality run #569 (`38955661858`) and Android Integration run #277 (`38955661789`) passed. Design is promising; **do not treat green UI tests as proof every route works.** Review-confirmed blockers/follow-ups: the Deposit quick action navigates to `AccountsScreen` rather than starting a deposit; "Edit Profile" routes to read-only `ProfileDetailsScreen`; "Clear Cache" only displays a fabricated "42 MB freed" success message without clearing anything; the Language selector only stores/syncs a preference and `MaterialApp.router` does not currently apply an app locale; the Settings row combines "Two-Factor & PIN" with a biometric-only status chip. Review the Help & FAQ claims (MoMo push, AZM ID transfer, escrow timing, withdrawal limits) against real code before shipping.
- These PRs were based on the same main SHA and must remain separate. No merge is implied by this planning note.

#### Current execution order and parallel ownership

1. **PR #155 settings wiring corrections.** Keep edits limited to Settings/Drawer/Support and their tests. Fix Deposit destination, real Edit Profile target, remove or implement the false Clear Cache action, make security status wording truthful, and verify Help & FAQ statements. Treat localisation as a real feature or clearly not available; do not leave a functioning-looking picker that only saves a value.
2. **PR #154 wallet interaction + Home composition.** Implement active selector toggle-to-tuck, label "Visa", stable typewriter-row geometry even when the current string is empty/typing/erasing, and measured reductions to the heading-to-cards gap. Preserve the same card header in both states, 44dp target, reduced motion, scroll-to-activity contract, and PIN gate. Capture viewport evidence at 320/360/390/412dp.
3. **Separate bottom-nav PR.** Make the pill content-sized instead of filling most of the width. Keep 44dp minimum hit areas. The selected tab displays its label: Home / Chat / Market; inactive tabs remain compact icons. Replace the pushed-page contextual band's ellipsis with an explicit "Switch tabs"/Tabs affordance, preserving its existing tab-switch behavior.
4. **Separate Marketplace Mix PR.** Make Mix an in-place mode within Marketplace, not a route push. Mix opens on All (unfiltered cross-category discovery); a specific category reveals its applicable taxonomy/subcategory row. Add a custom-drawn isometric die/cube and one-shot restrained growth/twist on activation (no new dependency, no perpetual animation, honor reduced motion). Add a subtle divider and spacing between the category dial and Mix. Make subcategories larger icon bubbles with descriptive category-aware icons. Preserve server order, exact-string GHS display and `payableIn` capability truth; do not fabricate prices or ranking.
5. **Read-only whole-app wiring audit, run in parallel.** Inventory every important button/setting/navigation item/CTA by screen, actual callback, route/API/provider, state mutation, tests, and evidence. Label each VERIFIED, PARTIAL, DEAD/MISROUTED, or UNSUPPORTED. The audit agent must not edit code; turn verified defects into small, owned PRs only after the report. No two code agents may own the same file set.
6. **Backend/operations safety lane continues in parallel.** Kotani Pay credentials/onboarding remain outstanding. Kotani is the agreed canonical transaction-rate provider, but the current composite CoinGecko+FX fallback can refresh the transaction-eligible canonical rate; implement a reviewed fail-closed Kotani-only canonical-rate change before real-money use. Firebase Admin initialization is live after backend PR #352, but real-device push delivery is not yet proven. Render currently has one free/single instance and no Redis URL; paid Render Key Value and distributed-mode proof are pre-onboarding gates. Keep Marketplace Discovery disabled until real business-confirmed GHS prices exist and the response is verified.
7. **Update this loop and the ledger at every material checkpoint.** Use exact full SHAs and CI run IDs. Don't claim a task completed because a PR exists; don't merge/deploy from an agent-only report.

#### Cross-chat continuity contract

A new ChatGPT conversation may not contain every prior transcript. This repository is the durable shared project brain; model memory is a helpful summary, not the implementation authority. On every new session, read `START_HERE.md`, this current block at the top of `ACTIVE_LOOP.md`, the current dated snapshot at the top of `CURRENT_STATE.md`, and current `EXECUTION_LEDGER.json`; then reconcile against current GitHub PRs, `main`, exact-head CI and Render before repeating work. Write down dates, what is verified, what is open, deferred decisions, the next single action and the evidence. Never create a competing tracker without updating this authority.


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
