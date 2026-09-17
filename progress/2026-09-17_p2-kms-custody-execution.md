# §P.2 KMS-capable custody execution landed (PR #277)

**Date:** 2026-09-17 · **Repo:** AzamanLTD/AZM-backend · **Branch:** `p2/kms-custody-execution` · **Squash:** `994f07c2e4fac60e1b20ea85889d9051eb143175` · **Exact head:** `11d37996885c9cde9ae487a6d2ea6f513ca4f09d` (exact-head CI green, run #1028)

## Explicitly distinguished states (the authoritative reconciliation record)

| State | Value |
|---|---|
| KMS-capable implementation landed | **TRUE** |
| Tatum provider contract implemented | **TRUE** |
| KMS/four-eye validation boundary implemented | **TRUE** |
| Production live signing/broadcast ACTIVE | **FALSE** |
| Real testnet execution exercised | **FALSE** |
| §P.3 custody accounting | **NOT STARTED** |

CI proof is deterministic mock-provider only: 72 custody proofs plus real-PostgreSQL proofs. No production financial data changed. Live signing/broadcast remains OFF unless separately authorized and configured.

## Architectural facts that landed

- **Canonical external execution boundary:** `services/tatumCustodyExecutionService.js` is the single canonical external crypto execution boundary; nothing else constructs Tatum transfer payloads. Sweep execution now uses the same custody boundary (`workers/onchainSweepWorker.js` → claim → submit → reconcile).
- **Asset identity:** native Polygon USDC only (`0x3c499c542cef5e3811e1192ce70d8cc03d5c3359`); bridged USDC.e is explicitly rejected. `WalletAddress` remains the canonical customer deposit-address authority from §P.1.
- **Money arithmetic:** exact integer base-unit internal arithmetic (BigInt); provider-facing decimal conversion via `baseUnitsToDecimalString` (exact strings, never through JS floating point); controller/worker authoritative writes use `Prisma.Decimal` derived from exact base-unit strings.
- **Durable execution record:** `CustodyExecution` persists kind, refId, network/asset/contract, from/to, exact base units, status, approvalStatus, `tatumPendingId`, txHash (only from real evidence), errorClass.
- **Submission:** atomic single-winner CAS claim (`RESERVING/REQUESTED → SUBMITTED` conditional update) before any provider call — a lost race converges, a crash-after-claim stays ambiguous, a second external submission is structurally impossible.
- **Lifecycle:** asynchronous signing/broadcast (`REQUESTED → RESERVING → SUBMITTED → SIGNING → BROADCAST → CONFIRMING → COMPLETED`), with `reconcilePendingExecutions` advancing via real provider/chain evidence; terminal states never move backward.
- **Unknown outcomes:** an HTTP timeout or crash-ambiguous submission remains `RECONCILIATION_REQUIRED` — no automatic refund after an ambiguous provider outcome (only a definitive 4xx `PROVIDER_REJECTED` refunds, atomically with the FAILED transition, exactly once).
- **No fabricated tx hashes:** a tx hash enters the record only from real provider/chain evidence (`isValidTxHash`-validated, unique). The KMS submission response carries only the prepared pending-transaction id (`tatumPendingId`), never a txId.
- **Settlement authority:** `COMPLETED` requires independently verified chain-transfer evidence (receipt success, Polygon, native-USDC contract, ERC-20 Transfer event, exact sender/recipient/base units); a matching-but-unrelated or reverted transaction never settles.
- **Provider contract:** Tatum token transfers use the current ERC-20 token endpoint (`POST /v3/blockchain/token/transaction`, `ChainTransferEthErc20KMS` schema — chain MATIC, exact decimal-string amount, digits 6, `signatureId`, `index` only for mnemonic-based IDs, no `from`); pending lifecycle via the documented KMS endpoints (list / complete-with-real-txId / delete). There is no fictional approve endpoint.
- **Four-eye:** documented external validation contract — the KMS daemon `GET`s `/api/internal/custody/kms/validate/:pendingId` and signs only on 2xx; the validator returns 2xx only for a durably APPROVED, exact-matching, non-terminal execution with a registry-verified signer. Mandatory on MAINNET (gate + preflight fail closed if disabled).
- **Signer identity:** no raw private-key execution path exists anywhere (KMS `signatureId` only, no silent fallback); the customer deposit signer and the master-hot signer are distinct separately-configured identities, verified against the ops-populated KMS signer registry (`tatum-kms getaddress` proofs), fail-closed in LIVE mode.
- **Production gates:** remain explicit and OFF unless separately authorized/configured (`TATUM_PROVIDER=LIVE` + `TATUM_KMS_ENABLED` + `TATUM_CRYPTO_EXECUTION_ENABLED`; the withdrawal endpoint returns `503 CRYPTO_EXECUTION_NOT_ENABLED` before any customer debit when gated off).

## The amendment (merged in the same PR, head `11d3799`)

The first version (head `dcca223`) was audited for contract faithfulness; 14 blockers were fixed before merge: the real Tatum request schema and decimal amount semantics, `tatumPendingId` field rename (fictional approve endpoint and unused columns removed), the four-eye externalUrl validator, the KMS signer registry replacing the xpub non-proof, a mainnet four-eye gate that actually blocks, the atomic single-winner CAS claim, evidence-branded settlement authority, and exact `Prisma.Decimal` money paths. Custody test count 57 → 72, zero regressions (the 11 failing full-suite suites are pre-existing localhost:5432 environment suites, identical on clean main). PR body documents the full amendment.

## Verification data

- PR #277 merged by Sugru 2026-09-17T17:13:47Z; squash `994f07c`, parent `639784d`.
- Exact head `11d3799`: CI `test` completed success (run #1028) — verified against GitHub.
- Post-merge main-head CI at `994f07c`: **completed success** (run 105304499068), verified against GitHub at reconciliation time (2026-09-17 ~17:30 UTC).
- Full local battery before merge: 72/72 custody suite on real PostgreSQL, route-check PASS, prisma validate, production audit 0 vulnerabilities, db-recovery-drill SUCCESS, overlays green.

## Scope boundary — §P.3 (NOT STARTED, do not implement yet)

§P.3 will introduce the formal: `CustodyAccount`, `CustodyMovement`, custody asset/liability accounting, evidence-backed Proof of Reserves, and treasury location/account semantics. None of those exist or are implemented by this merge; the §P.2 execution records will attach to that model when §P.3 starts. `docs/custody-execution.md` documents the deliberate scope boundary.

No backend production data was altered. No live signing/broadcast activation, no testnet execution, no §P.3 work is claimed by this record.
