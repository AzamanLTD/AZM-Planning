# Deposit-address authority: canonical WalletAddress registry (PR #275)

**Date:** 2026-09-17 · **Repo:** AzamanLTD/AZM-backend · **Branch:** `p0/wallet-address-authority` · **Squash:** `ea75f05` · **Exact head:** `4ce3de4` (CI green, 1402 tests)

## Problem

`User.tatumPolygonAddress` was a single mutable legacy string: no registry, no lifecycle, no uniqueness constraint, and owner resolution (`resolveOwner`) matched incoming deposit addresses against this mutable column. Two users could hold the same address string, a retired address could be handed to someone else, and bridged USDC.e was indistinguishable from native USDC at the deposit-identity level.

## Fix

- **Migration `20260917150000_wallet_address_authority` + boot installer `infra/install-wallet-address-overlay.js`** (idempotent: enum DO-block, `CREATE TABLE IF NOT EXISTS`, FK/index DO-blocks tolerant of `duplicate_object`, `ON CONFLICT DO NOTHING` backfill; derivationIndex = user.id is the deterministic derivation rule; the legacy column is NOT removed — it stays synced as a mirror for zero-downtime rollout).
- **`services/walletAddressService.js`** is the single authority: allocation, retirement, active-address query. DB-level guarantees: `(address, network)` globally unique; exactly one ACTIVE row per user+asset identity (partial unique index); RETIRED never reassigned; race-convergent allocation (losers re-read and return the winner).
- **Canonical identity:** native Polygon USDC (`0x3c499c542cef5e3811e1192ce70d8cc03d5c3359`); USDC.e / wrong-contract identities are rejected with no row created.
- **`resolveOwner`** reads the registry first; the deterministic legacy-mirror fallback is kept.

## Verification

- New real-PostgreSQL suite `wallet-address-governance.test.js` (13 cases), including an 8-way concurrent allocation race converging on one ACTIVE row and cross-owner global uniqueness enforced by the database itself.
- CI at `501134e` exposed 9 real-PG failures. All reproduced on a local Postgres 15 and root-caused before the fix commit `4ce3de4`:
  1. Backfill SQL referenced `User.updatedAt` — the column does not exist (User has `createdAt` only). `firstSeenAt/createdAt/updatedAt` now record `now()`, the truthful registration time; address-mint time is not derivable.
  2. The adoption path could INSERT a row for a legacy mirror address already owned by a RETIRED registry row (unique violation, rethrown). Adoption now pre-checks the registry: ACTIVE-for-user converges; any other owner/status derives a fresh address.
  3. Test fixture used a 43-char address against `VARCHAR(42)` (real Polygon addresses are `0x` + 40 hex).
  4. Per-test user deletion violated `TransactionHistory_userId_fkey` (seedUser backs seeded balances with ledger rows) — replaced with the established shift-suite afterEach `TRUNCATE "WalletAddress", "User" RESTART IDENTITY CASCADE`.
- Full jest suite verified **1402/1402 green locally** (Postgres 15, after applying the transaction-quote overlay — same provisioning as CI) before pushing `4ce3de4`.
- Exact-head CI green at `4ce3de4`; squash-merged as `ea75f05`.

## Environment note

The sandbox now runs a local Postgres 15 (`azm_test`, roles `test` and `postgres`, password `postgres`) enabling full-suite local verification before spending a CI cycle. Two suites depend on run order (rate-provenance-truthfulness, stale-rate-gate): run the full suite, not suites in isolation.
