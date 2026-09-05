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

### Verified current GitHub state — 2026-09-05 UTC

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e`; invoice creation/payment concurrency proofs are merged and verified with green exact-head tests and database recovery. Dine-in PR #233 is the executable invoice-to-closure orchestration proof: exact head `2453729846cf15d95bf6ab637d26ae19f3b735fa`, run `33957019469`, tests + production dependency audit + database recovery passed.
- Flutter main: `c750562d26499346e7c43315fba9912951e590d1`; PR #90 durable CLOSED-tab recovery is merged, and PR #91 adds the complementary customer payment convergence contract. Exact head `d1dbc94ed890583241a4d338fc2a045cf5cec4a3`, Flutter Quality run `33967426867`, passed Analyze + Test with coverage + upload coverage; merged at `c750562d26499346e7c43315fba9912951e590d1`.
- Business Portal main: `7140658f4d66cada3d6c3155c085638105fe484e`; PR #88 pointer palette insertion, PR #89/#91/#92/#94/#95 Wave A token/wiring slices, PR #93 bounded device-frame scroll, PR #96 Wave A completion, and PR #98 dine-in lifecycle invalidation proof are merged and verified. PR #96 exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf` passed smoke/tests/build; PR #98 exact head `85560129a19470e40a70175a8049eb3cffde8655`, Business Portal CI run `33975428983`, passed smoke/tests/build.
- Admin Portal main retains the verified withdrawal concurrency and financial API/settings boundary work; no dedicated dine-in projection was found in the current repository tree during the P0 audit.

### Studio acceptance gates

- **Wave A — VERIFIED / complete:** current main consumes grounded shared preview tokens across all 16 widget renderers plus action/fallback/selection/chrome geometry and the device frame. PR #96 adds an executable completion guard for zero inline numeric CSS `px` literals, renderer registry coverage, and frame tokenization. Keep this wave closed unless current Flutter measurements change.
- **Wave B — REOPENED / partial — 2D prerequisite decision made:** current V2 uses Pointer Events for palette insertion with capture, pointermove/up/cancel, before/after hit testing and click suppression. `StorefrontStudioV2.jsx` mounts the semantic layer tree + phone-preview stage, not the legacy `StorefrontCanvas.jsx` magnetic-snap surface. The legacy canvas still contains a coherent shared-token drag/snap implementation, but mounting it would create a second layout interaction model before V2 has explicit coordinate/persistence authority. **Do not reintroduce the historical 2D canvas just to satisfy old criteria.** A future 2D surface must first define coordinate/persistence authority and an executable geometry contract; only then should snap/fuse/settle be ported.
- **Wave C — REOPENED / partial:** current V2 has a bounded `studio-device-scroll-viewport` with vertical auto-scroll, horizontal clipping and overscroll containment around the transformed device preview. End-to-end rendered scroll, responsive relayout across device breakpoints and clipping evidence remain open.

### Dine-in P0 gate

- Backend PR #233 is merged and verified; no duplicate backend payment path or replacement proof is needed.
- Flutter PR #91 is merged at `c750562d26499346e7c43315fba9912951e590d1`. Its test locks the payment POST → authoritative tab reread → durable CLOSED recovery boundary and preserves the original failure when durable proof is absent.
- Business Portal PR #98 is merged at `7140658f4d66cada3d6c3155c085638105fe484e`; exact head `85560129a19470e40a70175a8049eb3cffde8655`, run `33975428983`, proves all supported DINE_IN_TAB_* lifecycle notifications invalidate canonical dine-in roots while unrelated order events do not.
- **P0 status: VERIFIED at cross-client contract scope.** Backend financial/replay authority, Flutter durable recovery and Business Portal owner-notification/query convergence each have current executable evidence. Socket payloads remain convergence signals only.
- **Residual integration gap:** no single deployed four-surface E2E harness currently correlates live Backend, Flutter, Business Portal and Admin state through one finalize/payment/replay/reconnect scenario. Admin-side dine-in lifecycle visibility also remains unproven because no dedicated dine-in projection was identified.

### Priority after current reconciliation

1. Exercise and, where justified, strengthen current Studio Wave C rendered evidence rather than reopening Wave B's deferred magnetic snap without a 2D surface.
2. Close the remaining integration gap in dine-in only when a testable deployed harness or an explicit Admin projection requirement makes that work valuable; do not duplicate existing component proofs.
3. Advance financial/control-plane integrity: withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
4. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence. Dine-in proof should be marked verified at the narrowest scope actually executed; do not promote component/contract proofs to a cross-client deployed-E2E claim.
