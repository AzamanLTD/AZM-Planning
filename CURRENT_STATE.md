# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC
**Authority:** current repository `main` + exact CI/evidence.

## Current verified baseline

- Backend main: `ad6110213f5a859fd9e47db75d0f36682c32974e` — security remediation, storefront retail catalog alignment, recovered dine-in payment notification hardening, executable invoice-to-closure orchestration proof, concurrent invoice payment replay proof, and concurrent invoice creation replay proof are merged and verified.
- Flutter main: `bf0589583522b44965a63379486ab33cb9d484e2` — canonical USDC/GHS FX convergence, CI modernization, durable dine-in payment recovery, and recovery parsing proof are merged and verified.
- Business Portal main: `62ceb099cafd1832cb45ba7cb14f14e18d4c2c1f` — Studio Wave A, Wave B, and Wave C device emulation are merged and verified. Wave C exact-head CI passed smoke, 148/148 tests, and production build.
- Admin Portal main: latest verified main includes withdrawal optimistic concurrency hardening plus the earlier financial API/settings boundary work.
- Planning main: reconciled with Backend PR #235 and all prior verified merges.

## Verified recent work

Backend PR #235 (`test: prove concurrent invoice creation replays committed invoice`) merged at `ad611021...` from exact tested head `d18e486f...`. Exact-head run #905 passed the full test suite and database backup/restore drill. The test proves two concurrent creation callers that both miss the preflight lookup converge through the durable unique idempotency-key race: one creates and one replays the same committed invoice.

Backend PR #234 (`test(invoice): prove concurrent payment replay safety`) merged at `205bfae0...` from exact tested head `97709b8d...`. Exact-head run #904 passed full tests and database backup/restore. It proves one atomic settlement and one durable payment replay without duplicate financial mutations.

Business Portal PR #86 (`feat(studio): add deterministic device emulation`) merged at `62ceb099...` from exact tested head `85bf12bb...`. It adds token-driven Phone/Tablet/Desktop stage emulation while preserving the measured Flutter `device.phone` contract and semantic responsive runtime model.

Backend PR #233 (`test(dine-in): prove invoice-to-closure orchestration`) merged at `526f659f...`, proving FINALIZED → invoice → PAID → CLOSED with tip propagation, deterministic `DINE_IN_TAB:<tabId>` idempotency, atomic closure and paid replay without recharging.

Backend PR #232 (`fix(dine-in): notify business on recovered payment`) merged at `7afbe1a7...`, ensuring durable recovered PAID/CLOSED state also emits `DINE_IN_TAB_PAID` for Business Portal convergence.

Flutter PR #90 (`test(dine-in): prove durable client payment recovery`) merged at `bf058958...`, accepting recovered payment only after authoritative reread proves the requested tab is CLOSED and rejecting wrong-tab, non-CLOSED and malformed responses.

## Active unresolved work

- Cross-client dine-in lifecycle still needs broader executable evidence across real client boundaries: duplicate/concurrent payment, tips, lost response, reconnect/background, multi-tab races, and Business/Admin convergence.
- Canonical business-invoice creation/payment idempotency now has executable concurrent proofs. Remaining caller discovery continues through repository-tree/file inspection because GitHub code search is unreliable; no concrete second production creation path has been identified in the inspected business routes.
- Financial/control-plane integrity is now the next engineering frontier: withdrawals beyond existing optimistic concurrency, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and load evidence.
- Production readiness remains future P0.

## Important discovered contracts

- Dine-in remains backend-authoritative: `OPEN → FINALIZED → CLOSED`; sockets are invalidation/convergence signals, while durable API state is authoritative.
- Recovered paid/closed dine-in state must notify Business Portal after durable closure.
- POS tax math is business-authoritative through transactional `BusinessTaxPreset` resolution and canonical tax computation; authoritative tax lines persist with the ledger entry.
- Storefront Studio is a semantic-tree editor. Wave A/B/C contain no AI generation, prompt compiler, free-form canvas, or localStorage persistence.
- Studio Wave B uses pointer capture, shared measured geometry, magnetic edge pull, logical edge-connected group fusion, continuous preview movement and persistence-time integer rounding.
- Studio Wave C keeps measured Flutter geometry separate from display presets and scales the existing renderer into deterministic Phone/Tablet/Desktop stage bounds.
- Retail collection is aligned across Flutter widget contract, Studio semantic/runtime adapter, Studio preview renderer, and backend catalog seed.
- Business invoice creation uses one canonical idempotency boundary with durable replay and intent-mismatch protection; concurrent unique-key races recover by rereading the committed invoice.
- Business invoice payment uses an atomic deterministic `INV_PAY_<invoiceId>` claim before any balance mutation and returns durable replay after a losing concurrent claim.

## Residual risks / hygiene

- Exhaustive invoice caller discovery should continue via repository-tree/file inspection until all relevant runtime creation paths are accounted for.
- `StorefrontPhonePreview.jsx` still has legacy CSS geometry outside the retail tokenized path; further tokenization should be measured and isolated.
- Existing merged backend branch deletion cleanup remains optional hygiene only if safe deletion tooling supports it.
- Bundle-size warning (>500 kB chunk) remains non-blocking and should be addressed as a dedicated performance slice.

## Next P0 sequence

1. Finish cross-client dine-in executable proof, emphasizing tips, lost-response/reconnect/background and Business/Admin authoritative convergence.
2. Advance financial/control-plane integrity across withdrawals, escrow disputes, fee controls, War Room, KPI accuracy, tenant/state/realtime and operational/load evidence.
3. Production readiness and adversarial/release rehearsal.
4. Studio legacy-geometry tokenization and bundle-size performance can proceed as isolated slices without displacing money/state authority work.

## Planning hygiene

Never promote failed or merely opened PRs into verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0.
