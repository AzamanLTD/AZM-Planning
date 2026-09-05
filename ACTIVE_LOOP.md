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
- Business Portal main: `83c1749e887f20846ec7af4876e5291816115ec7`; PR #88 pointer palette insertion, PR #89 first Wave A token slice, PR #91 second token-grounding slice, PR #92 renderer wiring, and PR #93 bounded device-frame scroll are merged and verified. PR #92 exact-head run #261 and PR #93 exact-head run #262 passed smoke, tests and build.
- PR #87 is closed/superseded; its exact-head fix run #252 passed, but the branch was over the change budget and none of that implementation counts as main.
- Admin Portal main retains the verified withdrawal concurrency and financial API/settings boundary work.

### Studio acceptance gates

- **Wave A — REOPENED / partial:** current main has verified tokenized slices for HeroHeader/QuickInfoBar/ProductGrid/ReviewCarousel/ContactCard and now Showcase/Location/Video, with the latter wired through shared token helpers. Remaining renderer and frame/chrome geometry must be tokenized in budgeted slices, with every token grounded against current Flutter source before completion.
- **Wave B — REOPENED / partial:** current V2 uses Pointer Events for palette insertion with capture, pointermove/up/cancel, before/after hit testing and click suppression. The historical 2D `StorefrontCanvas` magnetic snap/fuse/settle engine is not mounted by V2, so it is not to be reintroduced merely to satisfy a historical acceptance path; keep V2 semantic-tree interaction as the current authority.
- **Wave C — REOPENED / partial:** main now has a bounded `studio-device-scroll-viewport` around the transformed device preview with vertical auto-scroll, horizontal clipping and overscroll containment. End-to-end rendered scroll, responsive relayout and clipping evidence still need execution on current UI.

### Priority after Studio acceptance

1. Finish cross-client dine-in executable proof: tips, lost-response/reconnect/background recovery, multi-tab races, Business/Admin authoritative convergence.
2. Advance financial/control-plane integrity: withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
3. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence.
