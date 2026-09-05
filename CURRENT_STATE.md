# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `526f659f83af5d7fc708a0de964abad707d5a6a7` — security remediation, storefront retail catalog alignment, recovered dine-in payment notification hardening, and executable invoice-to-closure orchestration proof are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — canonical USDC/GHS FX convergence, CI modernization, durable dine-in payment recovery, and recovery parsing proof are merged and verified.
- Business Portal main: `62ceb099cafd1832cb45ba7cb14f14e18d4c2c1f` — Studio Wave A, Wave B, and Wave C device emulation are merged and verified. Wave C exact-head CI run #245 passed smoke, 148/148 tests, and production build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening.
- Planning main: reconciled with all verified merges through Business Portal PR #86.

## Verified recent work

Business Portal PR #86 (`feat(studio): add deterministic device emulation`) merged at `62ceb099...` from exact tested head `85bf12bb...`. It adds token-driven Phone/Tablet/Desktop stage emulation while preserving the measured Flutter `device.phone` contract and semantic responsive runtime model. The initial CI correctly caught a token contract regression; the implementation was corrected and the final exact-head run #245 passed all 148 tests and production build.

Backend PR #233 (`test(dine-in): prove invoice-to-closure orchestration`) merged at `526f659f...`, proving FINALIZED → invoice → PAID → CLOSED with tip propagation, deterministic `DINE_IN_TAB:<tabId>` idempotency, atomic closure and paid replay without recharging.

Backend PR #232 (`fix(dine-in): notify business on recovered payment`) merged at `7afbe1a7...`, ensuring durable recovered PAID/CLOSED state also emits `DINE_IN_TAB_PAID` for Business Portal convergence.

Flutter PR #90 (`test(dine-in): prove durable client payment recovery`) merged at `bf058958...`, accepting recovered payment only after authoritative reread proves the requested tab is CLOSED and rejecting wrong-tab, non-CLOSED and malformed responses.

## Active unresolved work

- Cross-client dine-in lifecycle still needs broader executable evidence across real client boundaries/read models: duplicate/concurrent payment, tips, lost response, reconnect/background, multi-tab races, and Business/Admin convergence.
- Canonical business-invoice caller audit remains incomplete only because exhaustive GitHub code search is unreliable; inspected creation paths use the canonical idempotency boundary and no concrete duplicate invoice math path has been identified.
- Financial/control-plane integrity remains the next engineering frontier: withdrawals beyond existing optimistic concurrency, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and load evidence.
- Production readiness remains future P0.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; sockets are invalidation/convergence signals, while durable API state is authoritative.
- Recovered paid/closed dine-in state must notify Business Portal after durable closure.
- POS tax math is business-authoritative through transactional `BusinessTaxPreset` resolution and canonical tax computation; authoritative tax lines persist with the ledger entry.
- Storefront Studio is a semantic-tree editor. Wave A/B/C contain no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Studio Wave B uses pointer capture, shared measured geometry, magnetic edge pull, logical edge-connected group fusion, continuous preview movement and persistence-time integer rounding.
- Studio Wave C keeps measured Flutter geometry separate from display presets and scales the existing 220px renderer into deterministic Phone/Tablet/Desktop stage bounds.
- Retail collection is aligned across Flutter widget contract, Studio semantic/runtime adapter, Studio preview renderer, and backend catalog seed.

## Residual risks / hygiene

- Exhaustive invoice caller discovery should continue via repository-tree/file inspection until all relevant runtime creation paths are accounted for.
- `StorefrontPhonePreview.jsx` still has legacy CSS geometry outside the retail tokenized path; further tokenization should be measured and isolated.
- Existing merged backend branch deletion cleanup remains optional hygiene only if safe deletion tooling supports it.
- Bundle-size warning (>500 kB chunk) remains non-blocking and should be addressed as a dedicated performance slice.

## Next P0 sequence

1. Finish cross-client dine-in executable proof, emphasizing duplicate/concurrent payment, recovery/reconnect/background and Business/Admin authoritative convergence.
2. Complete canonical business-invoice caller/idempotency audit via repository-tree inspection with evidence-driven changes only.
3. Advance financial/control-plane integrity across withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
4. Production readiness and adversarial/release rehearsal.

## Planning hygiene

Never promote failed or merely opened PRs into verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0.
