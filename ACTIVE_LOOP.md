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

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Verified current GitHub state — 2026-09-05 UTC

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e`; invoice creation/payment concurrency proofs are merged and verified with green exact-head tests and database recovery.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2`; durable dine-in recovery proof is merged and verified.
- Business Portal main: `62ceb099cafd1832cb45ba7cb14f14e18d4c2c1f`. Historical Studio Wave A/B/C implementation is merged, but acceptance is **REOPENED**.
- Business Portal PR #87 is the active Studio remediation slice; it is not verified or complete.
- Admin Portal main retains the verified withdrawal concurrency and financial API/settings boundary work.

### Studio acceptance gates

- **Wave A — REOPENED:** `StorefrontPhonePreview.jsx` renderer geometry is being routed through `toPreviewPx()` and shared tokens. Acceptance requires exact-head CI plus final token-grounding review against Flutter widget implementations and zero numeric inline pixel geometry in preview style objects.
- **Wave B — REOPENED / partial:** Studio V2 palette insertion now uses Pointer Events with capture and before/after hit testing. Acceptance still requires complete pointer capture/snap/fuse/settle executable evidence and no HTML5 drag surface.
- **Wave C — REOPENED / partial:** the phone frame is now bounded and scrollable using real overflow. Acceptance still requires executable proof of scrolling, responsive relayout, and demonstrable clipping/overflow behavior.

### Priority after Studio acceptance

1. Finish cross-client dine-in executable proof: tips, lost-response/reconnect/background recovery, multi-tab races, Business/Admin authoritative convergence.
2. Advance financial/control-plane integrity: withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
3. Production readiness and adversarial/release rehearsal.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success **and the wave acceptance list must be rechecked against current code before status becomes complete**.

## Planning synchronization

After every verified merge, reconcile `CURRENT_STATE.md`, this file and `EXECUTION_LEDGER.json`. Never write `complete` for a Studio wave unless every listed acceptance criterion is explicitly satisfied by current code plus matching executable evidence.
