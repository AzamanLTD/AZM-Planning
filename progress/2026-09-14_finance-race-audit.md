# 2026-09-14 finance P0 audit — current-main finding and verified remediation

## Finding
Current `AZM-backend/main` contained an unsafe read → decision → write boundary in `services/finance.service.js::liquidateProfits`.

The unsafe flow was:
1. ensure `SystemProfitFees` and `SystemFiatPool` singletons;
2. read `SystemProfitFees.balance`;
3. reject only when the pre-read was insufficient;
4. unconditionally decrement `SystemProfitFees.balance`;
5. increment `SystemFiatPool.balance`;
6. create `AdminProfitLog`.

Two concurrent liquidation requests could both pass the pre-read. The database's non-negative CHECK constraint prevented a persisted negative balance, but the losing transaction could fail through the database constraint rather than a stable domain-level insufficient-profit result.

## Remediation implemented
`AZM-backend` PR #238 replaced the pre-read + unconditional decrement with a single conditional claim:

```js
const claim = await tx.systemProfitFees.updateMany({
    where: { id: 1, balance: { gte: amountFloat } },
    data: { balance: { decrement: amountFloat } }
});

if (claim.count !== 1) {
    const err = new Error(`Insufficient profit balance. Requested: ${amountFloat.toFixed(6)} USDC.`);
    err.code = 'INSUFFICIENT_PROFIT_BALANCE';
    throw err;
}
```

The fiat-pool increment and `AdminProfitLog` write remain inside the same Prisma transaction. No duplicate controller/service authority was introduced.

## Regression coverage
Added `__tests__/finance-service-profit-liquidation-concurrency.test.js` covering:
- two concurrent liquidation attempts against one 10 USDC pool: exactly one succeeds;
- the losing attempt receives `INSUFFICIENT_PROFIT_BALANCE`;
- only one fiat-pool increment occurs;
- only one profit log is created;
- an oversized liquidation performs no fiat-pool or log mutation.

## Verification
- PR: `AzamanLTD/AZM-backend#238`
- Exact head verified by full CI: `f5d43f5ab321c2b00fa6814e08b1c1ed10fc58c5`
- Backend CI run `#916` passed all canonical steps: dependency install, production dependency audit, database schema application, Prisma generation, full Jest suite, route-registry verification, database backup/restore drill.
- Merged to backend main as `d5e96432214a6e5f369c7c59a0dba7b97ac7074c`.

## Follow-on tenant-isolation finding
During the same current-main audit, `services/businessOS/businessGroupService.js::getGroupStats(userId, groupId)` was found to filter by `groupId` alone when a group was supplied. This means a caller holding another owner's `groupId` could cause that group's businesses to be aggregated unless ownership is checked separately.

A canonical remediation is in progress in `AZM-backend` PR #239:
- verify `BusinessGroup.id` with `ownerUserId = userId` before reading;
- scope the subsequent `BusinessProfile.findMany` to both `groupId` and `userId`;
- fail closed with an empty stats result for a foreign/nonexistent group;
- add regression coverage for cross-owner rejection and valid-owner scoping.

This is a tenant-isolation P0 and is being handled separately from the finance transaction boundary so each change remains narrowly reviewable.

## Cleanup note
The earlier working branch `fix/finance-profit-liquidation-concurrency` is superseded by PR #238's canonical `fix/finance-profit-liquidation-concurrency-v2`. It contains no additional required implementation and should be removed when branch-deletion tooling is available.
