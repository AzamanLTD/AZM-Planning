# AZAMAN Active Engineering Loop

**Purpose:** persistent continuation state for long-running engineering work. Resume from here rather than restarting.

## Operating contract

1. Reconcile referenced repos, branches, PRs and SHAs before changes.
2. Research and trace the actual implementation before editing.
3. Execute the whole slice: implement → test → exact-head CI → diff audit → merge → verify main → reconcile Planning.
4. Never duplicate work already on current main.
5. CI failures are defects to diagnose and fix, not reasons to weaken tests.
6. While CI runs, perform independent audits instead of waiting idle.
7. After verified completion, advance immediately to the next unchecked P0.

## Current loop

**Status:** ACTIVE — DO NOT CLOSE

### Verified current GitHub state — 2026-09-05 UTC

- Backend main: `526f659f83af5d7fc708a0de964abad707d5a6a7`; PR #230 security, #231 retail catalog, #232 dine-in recovery notification, and #233 executable invoice-to-closure proof are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2`; PR #90 durable dine-in client recovery proof is merged and verified.
- Business Portal main: `62ceb099cafd1832cb45ba7cb14f14e18d4c2c1f`; Studio Wave A/B/C are merged and verified. PR #86 exact tested head `85bf12bb...` passed run #245: smoke, 148/148 tests, production build.
- Admin Portal main: latest verified withdrawal optimistic concurrency hardening remains on main.
- Planning main: reconciled through Studio Wave C merge.

### Active P0 work

#### 1. Cross-client dine-in lifecycle executable proof

Core server orchestration and Flutter durable recovery are proven. Remaining work is stronger executable cross-client evidence: duplicate/concurrent payment, tip propagation through all views, lost-response/reconnect/background recovery, multi-tab races, and authoritative Business/Admin convergence.

Existing Business Portal notification projection already invalidates legacy `openTabs` and `dineInTab` roots for `DINE_IN_TAB_*` events, so do not duplicate that implementation; strengthen missing executable behavior only.

#### 2. Canonical business-invoice caller/idempotency audit

Map remaining runtime invoice creation paths through repository-tree/file inspection because GitHub code search is unreliable. Preserve the existing canonical creation boundary, replay validation, tenant binding and canonical tax math. Implement only proven gaps.

#### 3. Financial/control-plane integrity

Advance through withdrawals, escrow disputes, fee profiles, War Room, KPI accuracy, tenant/state/realtime correctness and operational/load evidence. Prioritize money-moving/admin-authority mutations over cosmetic UI.

#### 4. Production readiness and adversarial verification

Prove configuration separation, secret handling/rotation, migration rollback discipline, observability, worker recovery, anomaly alerts, concurrency/load behavior and release rehearsal.

## Verified architectural contracts

- Dine-in authority is backend-owned: `OPEN → FINALIZED → CLOSED`.
- Socket messages are invalidation/convergence signals, never payment proof.
- Business Portal refetches authoritative API state after relevant notifications.
- Flutter accepts ambiguous payment outcomes as success only after durable reread proves the requested tab is `CLOSED`.
- POS tax authority comes from the business default `BusinessTaxPreset` through the canonical invoice calculation path; authoritative tax lines persist with ledger metadata.
- Studio is semantic-tree based. Wave A/B/C include no AI generation, prompt compiler, free-form canvas or localStorage persistence.
- Studio Wave C keeps measured Flutter geometry separate from display presets and uses token-driven phone/tablet/desktop stage emulation.

## Merge gate

No financial, tenant, state-machine or cross-repo authority change is complete until exact-head CI is green and required database recovery evidence passes, followed by verification on main. Studio changes require exact-head smoke/test/build success.

## Planning synchronization

After each verified merge, reconcile `CURRENT_STATE.md`, this file, and `EXECUTION_LEDGER.json` before selecting the next P0.
