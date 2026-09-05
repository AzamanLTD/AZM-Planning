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

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e`; invoice creation/payment concurrency proofs are merged and verified with green exact-head tests and database recovery. Dine-in PR #233 is the current executable invoice-to-closure orchestration proof: exact head `2453729846cf15d95bf6ab637d26ae19f3b735fa`, run `33957019469`, tests + production dependency audit + database recovery passed.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2`; durable dine-in recovery proof is merged and verified.
- Business Portal main: `2ef57f22b1dcae45334dd5f30e896e448073f10b`; PR #88 pointer palette insertion, PR #89/#91/#92/#94/#95 Wave A token/wiring slices, PR #93 bounded device-frame scroll and PR #96 Wave A completion are merged and verified. PR #96 exact head `fb20134bdd0ae622c8937e29c01f5e11a33a0abf` passed smoke, tests and production build; the completion guard verifies all 16 preview widget renderers and rejects inline numeric CSS pixel literals.
- PR #87 is closed/superseded; its exact-head fix run #252 passed, but the branch was over the change budget and none of that implementation counts as main.
- Admin Portal main retains the verified withdrawal concurrency and financial API/settings boundary work.

### Studio acceptance gates

- **Wave A — VERIFIED / complete:** current main consumes grounded shared preview tokens across all 16 widget renderers plus action/fallback/selection/chrome geometry and the device frame. PR #96 adds an executable completion guard for zero inline numeric CSS `px` literals, renderer registry coverage, and frame tokenization. Keep this wave closed unless current Flutter measurements change.
- **Wave B — REOPENED / partial — 2D prerequisite decision made:** current V2 uses Pointer Events for palette insertion with capture, pointermove/up/cancel, before/after hit testing and click suppression. `StorefrontStudioV2.jsx` mounts the semantic layer tree + phone-preview stage, not the legacy `StorefrontCanvas.jsx` magnetic-snap surface. Do **not** reintroduce the historical 2D canvas just to satisfy old criteria. A future 2D surface must first define its own coordinate/persistence authority and executable geometry contract; only then should snap/fuse/settle be ported.
- **Wave C — REOPENED / partial:** current V2 has a bounded `studio-device-scroll-viewport` with vertical auto-scroll, horizontal clipping and overscroll containment around the transformed device preview. End-to-end rendered scroll, responsive relayout across device breakpoints and clipping evidence remain open.

### Dine-in P0 gate

- Backend PR #233 is merged and verified; no duplicate backend payment path or replacement proof is needed.
- The next P0 work is the broader cross-client executable harness: Flutter → Backend → Business Portal → Admin, including finalize/payment/replay, tips, ambiguous response/reconnect/background recovery and authoritative business/admin convergence.

### Priority after current reconciliation

1. Execute the remaining cross-client dine-in proof rather than rewriting already-proven backend settlement logic.
2. Complete current Studio Wave C rendered evidence only after checking the actual V2 behavior.
3. Advance financial/control-plane integrity: withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
4. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence. Dine-in proof should be marked verified at the narrowest scope actually executed; do not promote a component-level proof to a cross-client E2E claim.
