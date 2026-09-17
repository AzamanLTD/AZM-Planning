# §P.1 closure-hardening: WalletAddress ownership + native-USDC webhook authority (PR #276)

**Date:** 2026-09-17 · **Repo:** AzamanLTD/AZM-backend · **Branch:** `p0/wallet-address-authority-closure` · **Squash:** `639784d` · **Exact head:** `e7cac0f` (CI green)

## Corrective findings (post-merge audit of PR #275)

1. **`body.userId` was a financial ownership authority.** `tatumCryptoWebhook` credited the payload-supplied `userId` directly, so a webhook caller could choose the credited user. Ownership now resolves ONLY through the WalletAddress registry from the actual deposit address + network; the legacy payload shape may still carry `userId`, but it is ignored for credit.
2. **`resolveOwner` returned RETIRED rows as active owners.** It now resolves ONLY an ACTIVE canonical row (network=POLYGON, asset=USDC, contract=`0x3c499c542cef5e3811e1192ce70d8cc03d5c3359`). If ANY registry row exists for an address, the registry state wins — the legacy `User.tatumPolygonAddress` mirror can never resurrect a retired address. History/audit is preserved via a separate `lookupWalletAddressHistory` (retirement removes financial eligibility, not the row).
3. **`USDC_E` constant was corrupted** (`0x2791b9717a737cA894...`, a mangled value). Corrected to the official Polygon USDC.e contract `0x2791bca1f2de4661ed88a30c99a7a9449aa84174`; exact-contract tests now pin both identities so they cannot drift again.
4. **The Tatum webhook accepted `asset: USDC.E` as canonical.** It is now rejected before any financial mutation. No contract-level inbound verification is claimed from the payload (it carries no token contract address) — that remains later custody-inbound work.

## Proofs (real PostgreSQL, `wallet-address-governance`, 18 cases)

- Address owned by A + payload `userId = B` → **A is credited, B is untouched** (balance + ledger-row assertions, not just response codes).
- Unknown address + arbitrary `userId` → **nobody is credited**; no `TransactionHistory` row, no balance change.
- `USDC.E` → ignored, zero mutation (no ledger row, both balances unchanged).
- Native `USDC` → still credited (positive control).
- RETIRED address → no active owner; the still-set mirror cannot resurrect it; history lookup still sees the row.
- Existing proofs all still green: production HMAC fail-closed (503/401), 8-way allocation race convergence, retired non-reuse, backfill idempotency, DB cross-owner uniqueness.
- **Suite hermeticity fix:** the webhook proofs credit the `SystemMasterCrypto`/`SystemHotWallet` singletons, which leaked into `withdrawal-fee-discount-atomicity`'s absolute-balance assertions in full-suite order. The suite's afterEach now truncates those singletons too.

## Completion gate (all green before the PR)

Focused suite 18/18; full backend suite **1407/1407** on real PostgreSQL 15; `prisma validate` clean; `route-check` PASS; `npm audit --omit=dev --audit-level=low` 0 vulnerabilities; repo-wide search confirms `services/walletAddressService.js` remains the sole writer of `User.tatumPolygonAddress`. Exact-head CI green at `e7cac0f`; squash-merged as `639784d`.

## Boundaries respected

No production financial data changed. No KMS, signing, broadcasting, custody accounting, treasury movement, exchange logic, Kotani on-ramp, or withdrawal-settlement changes. The onchain sweep LIVE branch remains a placeholder until §P.2 — which was NOT started.
