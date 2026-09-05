# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — invoice creation/payment concurrency proofs and prior financial/dine-in hardening are merged and verified. Backend PR #233 provides the executable dine-in invoice-to-closure proof; exact-head `2453729846cf15d95bf6ab637d26ae19f3b735fa`, run `33957019469`, passed tests, production dependency audit, and the database backup/restore drill before merge.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — durable dine-in recovery and prior FX/CI work are merged and verified.
- Business Portal main: `2ef57f22b1dcae45334dd5f30e896e448073f10b` — PR #88 pointer insertion, PR #89/#91/#92/#94/#95 Wave A token/wiring slices, PR #93 bounded device-frame scrolling, and PR #96 Wave A completion are merged and verified. PR #96 exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf`; exact-head CI passed smoke/tests/build and the PR stayed within the <=500 changed-line guard.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency and financial API/settings boundary work.

## Studio acceptance audit — current truth

Historical Wave A/B/C completion is not trusted merely because PR #86 had a green CI run. Each acceptance list is checked against current code and matching executable evidence before any wave is marked complete.

| Wave | Status | Code-verified evidence | Still required |
|---|---|---|---|
| A — Token foundation | **VERIFIED / complete** | Current main wires all 16 storefront catalog widget renderers through the shared preview token source; remaining Animated Counter, Custom HTML, Gradient Hero, action/fallback/selection/chrome and phone-frame geometry were completed in PR #96. The executable Wave A completion guard rejects inline numeric CSS `px` literals, rejects direct numeric `px(...)` calls, verifies all 16 registry keys, and verifies tokenized frame dimensions. Exact-head PR #96 CI `fb20134bdd0ae622c8937e29c01f5e11a33a0abf` passed smoke/tests/build. | None for Wave A. Keep current acceptance guard in place; do not reopen for historical renderer churn unless current Flutter measurements change. |
| B — Pointer drag with magnetic snap | **REOPENED / partial — 2D prerequisite decision made** | PR #88 merged Pointer Events palette insertion with capture, pointermove/up/cancel cleanup, before/after hit testing and click suppression. Current V2 is a semantic layer-tree + phone-preview editor. `StorefrontCanvas.jsx` still contains the historical 2D magnetic snap/fuse/settle engine, but V2 does not mount it. | Do not port/reintroduce legacy 2D snap behavior yet. Current V2 has no intentional free-form 2D coordinate editing surface; palette insertion is the active V2 drag acceptance. Restore/define a V2 2D surface first (coordinate model, persistence authority, collision/snap contract, executable geometry tests), then revisit magnetic snap/fuse/settle as a separate bounded slice. |
| C — Real device emulation | **REOPENED / partial** | PR #93 merged a dedicated bounded emulator scroll viewport with `overflowY:auto`, horizontal clipping, overscroll containment and executable contract tests. Device dimensions/scaling remain tokenized. | Exercise end-to-end scrolling with overflowed preview content, verify responsive relayout across phone/tablet/desktop, and prove clipping/overflow behavior against rendered UI. |

## Dine-in P0 proof — current truth

- The requested backend settlement proof already exists on current main; do not create duplicate payment logic or a second test package.
- Backend PR #233 (`test(dine-in): prove invoice-to-closure orchestration`) is merged and verified. Its test proves `FINALIZED → invoice creation → PAID → CLOSED` with tip propagation, deterministic `DINE_IN_TAB:<tabId>` invoice identity, atomic tab transition, customer settlement socket signal, and durable PAID/CLOSED replay without calling payment again.
- Exact-head CI run `33957019469` on `2453729846cf15d95bf6ab637d26ae19f3b735fa` passed the full test job; that job explicitly passed production dependency audit, tests, and the database backup/restore drill. Current `main` still contains `__tests__/dine-in-lifecycle-orchestration.test.js`.
- Cross-client executable proof remains a broader residual: Flutter recovery proof is merged in PR #90, Business Portal authoritative invalidation/convergence exists, but the full Flutter → Backend → Business Portal → Admin end-to-end harness has not been demonstrated as one executable scenario.

## Active Studio implementation

- PR #87 is **closed/superseded**, not merged. Its exact-head fix commit was `e62b6c7ae99ca8771ad3ee1e177d4321f3c1ec73`; exact-head run #252 passed, but the PR exceeded the normal per-PR budget and remains reference material only.
- PR #88 is merged at `76e39eb6ba937082684cc72189c34f7157b967a8`; exact-head run #253 passed smoke/tests/build.
- PR #89 is merged at `59d9567d72b4eb0798a1a97f2fc9877381725a26`; exact head `cc37a2058b8c68ba1ddcbe6663f86e1474af110d3` and run #255 passed smoke/tests/build.
- PR #91 is merged at `a1002e0846f7160298dc58e3f8261e675a9a1c5f`; exact head `df1bcd0d2f6903d396bc95affec9311b306fbfa9` and run #258 passed smoke/tests/build.
- PR #92 is merged at `fc4536492c2c879ad2ff2b47aefe5c6c83dcc226`; exact head `08955de01a48bcbcac3e621c7741e2781a3d4843` and run #261 passed smoke/tests/build. It wired Showcase Gallery, Location Map and Video Player to the grounded token slice.
- PR #93 is merged at `83c1749e887f20846ec7af4876e5291816115ec7`; exact head `0952e426dfcf451ebfecd51c8ea3ee9afa2d8a16` and run #262 passed smoke/tests/build. It adds the bounded device-frame scroll boundary.
- PR #94 is merged at `f5e8470898b87462d9febb152e0d8faafef4569e`; its exact-head CI passed and it wired Promo Banner, Social Feed and Live Stats.
- PR #95 is merged at `c64a2f78e68781402beb7a89ec90340097ac9503`; exact-head CI passed and it grounded the remaining Animated Counter, Custom HTML and Gradient Hero token maps.
- PR #96 is merged at `2ef57f22b1dcae45334dd5f30e896e448073f10b`; exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf` passed exact-head CI and completed current Wave A token consumption/guardrails.

## Important contracts

- Dine-in authority remains backend-owned: `OPEN → FINALIZED → CLOSED`.
- Business invoice creation and payment use deterministic durable idempotency anchors with replay rather than duplicate money movement.
- Studio remains semantic-tree based with no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Before any Studio wave is written as **complete**, inspect the acceptance list, inspect current code, execute or cite matching tests/CI, and record every residual gap.

## Residual risks / hygiene

- Wave C has a real bounded scroll viewport on main, but end-to-end scroll, responsive relayout and rendered clipping evidence are still outstanding.
- Current V2 does not mount the legacy 2D canvas. This is an intentional architecture decision for now, not an unaddressed duplicate implementation; any future 2D surface must define its own semantic coordinate/persistence authority before snap is ported.
- Cross-client dine-in end-to-end execution (single harness spanning Flutter, Backend, Business Portal, Admin) remains outstanding even though the core backend and Flutter recovery proofs are merged.
- Bundle-size warning (>500 kB chunk) remains a separate performance slice.
- Backend branch-deletion cleanup remains optional hygiene only where safe tooling supports it.
