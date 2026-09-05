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

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e`; invoice creation/payment concurrency proofs are merged and verified with green exact-head tests and database recovery.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2`; durable dine-in recovery proof is merged and verified.
- Business Portal main: `a1002e0846f7160298dc58e3f8261e675a9a1c5f`; PR #88 pointer palette insertion, PR #89 first Wave A token slice, and PR #91 second Flutter-grounded token slice are merged. PR #91 exact-head run #258 passed smoke, tests and build.
- PR #87 is closed/superseded; its exact-head fix run #252 passed, but the branch was over the change budget and none of that implementation counts as main.
- Admin Portal main retains the verified withdrawal concurrency and financial API/settings boundary work.

### Studio acceptance gates

- **Wave A — REOPENED / partial:** the preview must use shared token data routed through `toPreviewPx()` with zero numeric inline pixel literals in preview style objects. Current main has a verified first slice (HeroHeader, QuickInfoBar, ProductGrid, ReviewCarousel, ContactCard) and a second grounded token slice (Showcase Gallery, Location Map, Video Player). Those second-slice tokens still need renderer wiring. Remaining renderer and frame/chrome geometry must be tokenized in budgeted slices, with every token grounded against current Flutter source before completion.
- **Wave B — REOPENED / partial:** palette insertion is now Pointer Events on main with capture, pointermove/up/cancel, before/after hit testing and click suppression. Acceptance still requires rechecking the historical magnetic snap/fuse/settle criteria against current Studio V2 code with executable evidence.
- **Wave C — REOPENED / partial:** current main retains device emulation tokens/stage but does not yet have the unmerged PR #87 scroll implementation. Acceptance requires a bounded real-overflow phone frame, executable scroll proof, responsive relayout, and demonstrable clipping/overflow.

### Priority after Studio acceptance

1. Finish cross-client dine-in executable proof: tips, lost-response/reconnect/background recovery, multi-tab races, Business/Admin authoritative convergence.
2. Advance financial/control-plane integrity: withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
3. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence.
