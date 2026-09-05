# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — invoice creation/payment concurrency proofs and prior financial/dine-in hardening are merged and verified.
- Flutter main: `c750562d26499346e7c43315fba9912951e590d1` — durable dine-in recovery plus the new payment convergence contract are merged and verified.
- Business Portal main: `2ef57f22b1dcae45334dd5f30e896e448073f10b` — PR #88 pointer palette insertion, PR #89/#91/#92/#94/#95 Wave A token/wiring slices, PR #93 bounded device-frame scrolling, and PR #96 Wave A renderer/chrome completion are merged and verified.
- Admin Portal main: latest verified main retains withdrawal optimistic concurrency and financial API/settings boundary work; the current tree contains no dedicated dine-in projection identified during the P0 audit.

## Studio acceptance audit — current truth

Historical Wave A/B/C completion is not trusted merely because an older green run exists. Each acceptance list is checked against current code and matching executable evidence before completion is recorded.

| Wave | Status | Code-verified evidence | Still required |
|---|---|---|---|
| A — Token foundation | **VERIFIED / complete** | PR #96 exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf` merged at `2ef57f22b1dcae45334dd5f30e896e448073f10b`; exact CI passed smoke, tests, and production build. Current preview consumes grounded shared tokens across all 16 catalog renderers plus action/fallback/selection/chrome/frame geometry. The executable completion guard verifies all 16 renderers and rejects numeric inline CSS `px` literals. | None under the current Flutter measurements. Reopen only if the source geometry contract changes. |
| B — Pointer drag with magnetic snap | **REOPENED / partial — 2D prerequisite decision made** | PR #88 merged Pointer Events palette insertion with capture, pointermove/up/cancel cleanup, before/after hit testing and click suppression. Current `StorefrontStudioV2.jsx` mounts the semantic layer tree + phone-preview stage, not the legacy 2D `StorefrontCanvas.jsx` magnetic-snap surface. | Keep palette insertion as the current V2 drag acceptance. **Do not reintroduce the historical 2D canvas solely to satisfy old criteria.** Magnetic snap/fuse/settle stays deferred until a deliberate V2 2D surface defines coordinate ownership, persistence semantics, collision/geometry contracts, and executable tests. |
| C — Real device emulation | **REOPENED / partial** | PR #93 merged a bounded `studio-device-scroll-viewport` with vertical auto-scroll, horizontal clipping and overscroll containment around the transformed device preview. Device dimensions/scaling are tokenized. | Exercise rendered end-to-end scroll with genuinely overflowing content; prove phone/tablet/desktop responsive relayout and clipping/overflow behavior against rendered UI. |

## Active Studio implementation

- PR #87 is **closed/superseded**, not merged. Exact-head fix commit `e62b6c7ae99ca8771ad3ee1e177d4321f3c1ec73`; exact-head run #252 passed, but the PR exceeded the normal change budget.
- PR #88 merged at `76e39eb6ba937082684cc72189c34f7157b967a8`; exact-head run #253 passed smoke/tests/build.
- PR #89 merged at `59d9567d72b4eb0798a1a97f2fc9877381725a26`; exact head `cc37a2058b8c68ba1ddcbe6663f86e1474af110d3`, run #255 passed.
- Historical Business Portal PR #91 is the second token-map slice; it merged at `a1002e0846f7160298dc58e3f8261e675a9a1c5f`; exact head `df1bcd0d2f6903d396bc95affec9311b306fbfa9`, run #258 passed.
- PR #92 merged at `fc4536492c2c879ad2ff2b47aefe5c6c83dcc226`; exact head `08955de01a48bcbcac3e621c7741e2781a3d4843`, run #261 passed. It wires Showcase Gallery, Location Map and Video Player to grounded tokens.
- PR #93 merged at `83c1749e887f20846ec7af4876e5291816115ec7`; exact head `0952e426dfcf451ebfecd51c8ea3ee9afa2d8a16`, run #262 passed. It adds the bounded device-frame scroll boundary.
- PR #94 merged at `f5e8470898b87462d9febb152e0d8faafef4569e`; exact-head CI passed. It grounded/wired Promo Banner, Social Feed and Live Stats.
- PR #95 merged at `c64a2f78e68781402beb7a89ec90340097ac9503`; exact-head CI passed. It added the remaining Animated Counter, Custom HTML and Gradient Hero token maps.
- PR #96 merged at `2ef57f22b1dcae45334dd5f30e896e448073f10b`; exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf`, exact CI passed smoke/tests/build. This closes current Wave A implementation and acceptance.

## Dine-in P0 — current evidence and residual proof

- Backend is authoritative for `OPEN → FINALIZED → CLOSED`; the canonical invoice service remains the financial authority.
- Backend PR #232 merged at `7afbe1a7318402519aafed6142fdb7364f9df86d` with exact-head `186ac370bc75c60d5980a0bb8c3d308f812b94f0` and production audit/tests/database recovery passing; recovered paid settlement also emits the business-owner lifecycle notification.
- Backend PR #233 merged at `526f659f83af5d7fc708a0de964abad707d5a6a7` with exact-head `2453729846cf15d95bf6ab637d26ae19f3b735fa`, run `33957019469`, and tests/production audit/database recovery passing. It proves FINALIZED → invoice → PAID → CLOSED with tip and deterministic `DINE_IN_TAB:<tabId>` idempotency, plus durable PAID/CLOSED replay without a second payment mutation.
- Backend PR #234 merged at `205bfae0082303253638c40828df9928f820d7cb` with exact-head `97709b8d5d42494c460997dd0053eabfc7a2dea6`, proving concurrent payment claim/replay safety.
- Backend PR #235 merged at `ad6110213f5a859fd9e47db75d0f36682c32974e` with exact-head `d18e486fb1bd3bd07f2c3973113e9a422c9c0a1d`, proving concurrent invoice creation converges on the same idempotent invoice.
- Flutter PR #90 merged into `bf058958...` with exact CI `e479e07546d723afb48d6d841ebfecde452b30f8`, proving the durable CLOSED-tab recovery parser.
- Flutter PR #91 merged at `c750562d26499346e7c43315fba9912951e590d1`; exact head `d1dbc94ed890583241a4d338fc2a045cf5cec4a3`, Flutter Quality run `33967426867` passed Analyze/Test with coverage. The added contract verifies payment POST → authoritative tab reread → durable CLOSED recovery, while preserving the original failure when durable proof is absent.
- Business Portal already persists/emits owner-scoped `DINE_IN_TAB_*` lifecycle notifications and invalidates `dine-in`, `dine-in-tabs`, `openTabs`, and `dineInTab`; socket payloads remain invalidation signals and canonical API reads remain authoritative.
- **Residual P0 gap:** current evidence is a coordinated set of executable component/contract proofs, not a single deployed live four-surface E2E run. Admin-side lifecycle visibility also remains unproven because no dedicated dine-in projection was identified in the current Admin tree.

## Important contracts

- Dine-in financial authority remains backend-owned and uses deterministic durable idempotency/replay rather than duplicate money movement.
- Studio remains semantic-tree based with no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Current Wave B decision is to avoid legacy 2D restoration solely for historical snap criteria.
- Planning completion claims require current-code evidence plus matching executable tests/CI; historical green runs alone never close a wave.

## Residual risks / hygiene

- Wave C rendered scroll, responsive relayout and clipping evidence remain open.
- Cross-client dine-in P0 still needs a stronger executable harness and explicit Admin visibility proof before it can be called end-to-end complete.
- Bundle-size warning (>500 kB chunk) remains a separate performance slice.
- Backend branch-deletion cleanup remains optional hygiene only where safe tooling supports it.
