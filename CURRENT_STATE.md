# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — invoice creation/payment concurrency proofs and prior financial/dine-in hardening are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — durable dine-in recovery and prior FX/CI work are merged and verified.
- Business Portal main: `83c1749e887f20846ec7af4876e5291816115ec7` — PR #88 pointer insertion, PR #89 first Wave A renderer-token slice, PR #91 second Flutter-grounded token slice, PR #92 renderer wiring, and PR #93 bounded device-frame scrolling are merged and verified.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency and financial API/settings boundary work.

## Studio acceptance audit — current truth

Historical Wave A/B/C completion is not trusted merely because PR #86 had a green CI run. Each acceptance list is checked against current code and matching executable evidence before any wave is marked complete.

| Wave | Status | Code-verified evidence | Still required |
|---|---|---|---|
| A — Token foundation | **REOPENED / partial** | PR #89 merged the first tokenized renderer slice; PR #91 added Flutter-grounded Showcase/Location/Video token maps; PR #92 wired those maps into their renderers and added executable no-inline-geometry coverage. | Tokenize the remaining renderer set and frame/chrome geometry, ground every remaining token against current Flutter source, remove remaining numeric inline pixel geometry from preview style objects, then pass a complete criterion-by-criterion audit with exact-head CI. |
| B — Pointer drag with magnetic snap | **REOPENED / partial** | PR #88 merged Pointer Events palette insertion with capture, pointermove/up/cancel cleanup, before/after hit testing and click suppression. Current V2 is a semantic layer-tree + phone-preview editor; the historical 2D `StorefrontCanvas` magnetic snap engine is not mounted by V2. | Keep palette insertion as the current V2 drag acceptance. Do not reintroduce the legacy 2D canvas solely to satisfy historical criteria. Revisit magnetic snap/fuse/settle only if a current V2 2D canvas surface is intentionally restored. |
| C — Real device emulation | **REOPENED / partial** | PR #93 merged a dedicated bounded emulator scroll viewport with `overflowY:auto`, horizontal clipping, overscroll containment and executable contract tests. Device dimensions/scaling remain tokenized. | Exercise end-to-end scrolling with overflowed preview content, verify responsive relayout across phone/tablet/desktop, and prove clipping/overflow behavior against rendered UI. |

## Active Studio implementation

- PR #87 is **closed/superseded**, not merged. Its exact-head fix commit was `e62b6c7ae99ca8771ad3ee1e177d4321f3c1ec73`; exact-head run #252 passed, but the PR exceeded the normal per-PR budget and remains reference material only.
- PR #88 is merged at `76e39eb6ba937082684cc72189c34f7157b967a8`; exact-head run #253 passed smoke/tests/build.
- PR #89 is merged at `59d9567d72b4eb0798a1a97f2fc9877381725a26`; exact head `cc37a2058b8c68ba1ddcbe6663f86e1474af110d3` and run #255 passed smoke/tests/build.
- PR #91 is merged at `a1002e0846f7160298dc58e3f8261e675a9a1c5f`; exact head `df1bcd0d2f6903d396bc95affec9311b306fbfa9` and run #258 passed smoke/tests/build.
- PR #92 is merged at `fc4536492c2c879ad2ff2b47aefe5c6c83dcc226`; exact head `08955de01a48bcbcac3e621c7741e2781a3d4843` and run #261 passed smoke/tests/build. It wires Showcase Gallery, Location Map and Video Player to the grounded token slice.
- PR #93 is merged at `83c1749e887f20846ec7af4876e5291816115ec7`; exact head `0952e426dfcf451ebfecd51c8ea3ee9afa2d8a16` and run #262 passed smoke/tests/build. It adds the bounded device-frame scroll boundary.
- Next Studio work must remain <=500 changed lines per PR with at least one test. The active Wave A objective is now the remaining renderer/chrome tokenization, not replaying #87.

## Important contracts

- Dine-in authority remains backend-owned: `OPEN → FINALIZED → CLOSED`.
- Business invoice creation and payment use deterministic durable idempotency anchors with replay rather than duplicate money movement.
- Studio remains semantic-tree based with no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Before any Studio wave is written as **complete**, inspect the acceptance list, inspect current code, execute or cite matching tests/CI, and record every residual gap.

## Residual risks / hygiene

- Wave A still has remaining renderers and shared frame/chrome geometry with numeric inline pixels on current main.
- Wave A token values not directly grounded in the current Flutter widget source remain unacceptable as completion evidence.
- Wave C has a real bounded scroll viewport on main, but end-to-end scroll, responsive relayout and rendered clipping evidence are still outstanding.
- Bundle-size warning (>500 kB chunk) remains a separate performance slice.
- Backend branch-deletion cleanup remains optional hygiene only where safe tooling supports it.
