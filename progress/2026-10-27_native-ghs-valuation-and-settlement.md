# Native-GHS valuation + settlement wave (2026-10-27, verified)

Owner intent (#369): GHS balances are authoritative, USDC is the estimate; the
intended native-GHS settlement integration ships flag-gated OFF with zero live
behaviour change. All work in AzamanLTD/AZM-backend. Nothing merged, nothing
activated.

## PR #369 — valuation contract correction (head `9909fe2`)

- Headline = authoritative native-GHS balance ONLY. No ≈, no FX folded in, no USDC.
- USDC equivalent beneath it as the estimate: `GHS / rate`. A missing FX rate
  no longer nullifies the GHS headline.
- Owner regression pinned: 100 GHS + 500 USDC @ rate 10 → headline `100 GHS`, estimate `10 USDC`.
- 91/91 valuation suites green; both CI lanes green at `9909fe2`
  (test run 40572520451, financial-durability run 40572520452).

## PR #381 — native-GHS settlement, collections leg (head `8dcfaae`, base = #369 branch)

- Flag: `GlobalSettings.nativeGhsSettlementEnabled` (default FALSE), installed by
  `infra/install-native-ghs-settlement-overlay.js`, wired into `npm run release` and all 3 CI chains.
- OFF: settlement byte-identical to legacy USDC conversion credit (dedicated flag-OFF regression).
- ON: settled Moolre collections credit `nativeGhsBalance` in exact pesewas;
  `availableBalance` never touched; §P.4 posts `D fiat:momo:ghs / C ghs:user:{id}:liability`;
  Model B + `clearing:conversion` skipped; both flags on fails CLOSED before any mutation.
- Durability: deposit ONCE stays with the PENDING→COMPLETED CAS; ledger key
  `ledger:deposit:fiat-ghs:{ref}`; once-only reversal via raw jsonb CAS stamp on
  `metadata.nativeGhsReversal` (Prisma JSON `equals: null` misses absent keys — a
  real audit finding); `reconcileNativeGhsSettlements` flags stranded GHS_NATIVE rows as
  ReconciliationExceptions, never mutates money.
- Tests: 7/7 new real-PostgreSQL suite + 232/232 affected battery locally; both CI lanes
  green at `8dcfaae` (test run 40574078315, financial-durability run 40574078812).
- Settled rows record honest identity markers: `settlementCurrency` + `settlementExternalRef`
  (the exact ledger-key identity — externalRef ≠ txHash when the provider carries its own
  relatedReference).

## Open decision (owner)

GHS-native WITHDRAWAL leg not implemented: whether a GHS-native MoMo payout skips the
quote/FX path entirely, and what rate governs its pool settlement. Recorded on #369 and in
AZM-backend docs/native-ghs-settlement.md. Blocks any flag activation.

## Frontend dependency note

PR #203 (AZM-frontend balance card) already implements the matching display grammar:
GHS dominant headline, USDC estimate with ≈ on the USDC line only. Contract aligned.
