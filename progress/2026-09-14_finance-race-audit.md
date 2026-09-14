# 2026-09-14 finance P0 audit — current-main finding

## Finding
Current `AZM-backend/main` contains an unsafe read → decision → write boundary in `services/finance.service.js::liquidateProfits`.

Current flow:
1. ensure `SystemProfitFees` and `SystemFiatPool` singletons;
2. read `SystemProfitFees.balance`;
3. reject only when the pre-read is insufficient;
4. unconditionally decrement `SystemProfitFees.balance`;
5. increment `SystemFiatPool.balance`;
6. create `AdminProfitLog`.

Two concurrent liquidation requests can both pass the pre-read. The database's non-negative CHECK constraint prevents a persisted negative balance, but the losing transaction currently fails through the database constraint rather than a stable domain-level insufficient-profit result. The read/decision/write boundary therefore remains weaker than the platform's required atomic-claim standard.

## Evidence
- Backend main baseline before CI-hardening: `94dfd3144fc3abfd60f51e7a38ad241809b3e8e2`.
- `SystemProfitFees` and `SystemFiatPool` both have DB-level non-negative balance constraints in `prisma/migrations/20260525_phase_j2_balance_check_constraints/migration.sql`.
- Existing `processFiatWithdrawal` already demonstrates the canonical conditional-claim pattern using `updateMany` with `{ balance: { gte: amount } }` and a stable `FIAT_POOL_INSUFFICIENT` error.
- Existing liquidation route remains a single controller → `financeService.liquidateProfits` path; no duplicate financial service was introduced.

## Required next implementation
Replace the liquidation pre-read + unconditional decrement with one conditional atomic balance claim and explicit domain error semantics, then prove:
- successful liquidation moves exactly the requested amount;
- concurrent loser cannot create a second liquidation/profit log or overdraw SystemProfitFees;
- Fiat pool increment and profit debit remain one transaction;
- Admin response maps insufficient-profit contention to a stable error without exposing raw Prisma constraint failures.

Do not revive stale withdrawal/liquidation branches. Implement against current main and keep the PR <=500 changed lines with executable regression coverage.

## Self-audit status
This finding was not patched through a controller wrapper or parallel service because that would create a second authority. The dedicated branch `fix/finance-profit-liquidation-concurrency` currently exists as a working branch with no merged production changes and should be either completed through the canonical service or removed when branch-deletion tooling is available.
