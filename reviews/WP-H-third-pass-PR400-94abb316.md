# WP-H third pass — PR #400 consumer wiring (reviewed 94abb316, head had moved to c703e14f)

Agent 2, read-only. Supersedes nothing: the `6a6f233` and `c14b6a13` reports above stand.

## Identity

- Requested: `94abb3166119312a3f126ca968ae085d8b3dcaf2`
- Actual remote head at review time: `c703e14facd043d78b747b141bb75d23b9213dd6` (one commit ahead: "fix(authority): the pool reader publishes one locked snapshot, and a lock failure is UNKNOWN")
- `94abb316` is an ancestor of the head; both were read. Findings below state which SHA each applies to.
- Executed by me: `__tests__/wp-h-tristate-authority.pg.test.js` on a disposable PostgreSQL 15.19 built from the candidate's own release sequence → **19/19, exit 0**.

## 1. `readModeAndPool()`

**At `94abb316` — DEFECT (High).** `requireAuthorityState(db)` resolves the state, then a **separate** `$queryRawUnsafe` reads `availableGhs`/`projectionBalance`. No transaction, no lock. The unit and the number come from two different instants.

Scenario: resolve → `ENFORCED_INACTIVE` (unit USDC) → activation commits between the two statements (exclusive lock, projection synced to 0, `availableGhs` set) → the second query reads `projectionBalance = 0` → an admin/profit/finance reader publishes **"USDC 0.00"** while the live regime is GHS with liquidity. The reverse interleave is not reachable (the record is append-only), so the failure is stale-unit under-reporting rather than GHS-as-USDC, but it is still a wrong published balance.

**At `c703e14f` — FIXED, and I verified the part that could have been fake.** The reader now runs `db.$transaction((tx) => withAuthoritySnapshot(tx, read), { timeout: 20000 })` (`src/services/fiatLiquidityActivationService.js:182-183`), so `pg_advisory_xact_lock_shared` is taken **inside** an interactive transaction and held until commit. That distinction matters: an advisory *xact* lock issued on a bare Prisma client is released when that implicit transaction ends, so a helper-only test would pass while providing no exclusion. The call site genuinely opens the transaction.

`withAuthoritySnapshot` also converts a lock-acquisition failure into `AUTHORITY_STATE_UNAVAILABLE` (`AUTHORITY_LOCK_UNAVAILABLE`) rather than a raw driver error — correct, since a caller must not retry into a guess.

## 2. `payoutBatchWorker`

**Open defect at both SHAs (High).** The UNKNOWN guard runs at `_processBatch()` entry as `await isAuthorityModeActive(this.prisma)` — an unlocked, non-transactional read. The row claim is at `workers/payoutBatchWorker.js:550-556`:

```js
const claim = await this.prisma.withdrawal.updateMany({
    where: { id: withdrawal.id, status: { in: ['PENDING', 'APPROVED'] } },
    data: { status: 'PROCESSING' }
});
```

Bare client, no transaction, no authority re-read, and `grep` finds **no** `lockAuthorityShared` / `withAuthoritySnapshot` anywhere in the worker.

So activation can commit between the batch-level decision and the per-row claim, and the row is then claimed and dispatched under a regime that has already ended. Nothing database-enforced closes it: the CAS (`count !== 1` → `WITHDRAWAL_ALREADY_CLAIMED`) serialises **workers against each other**, not regimes; and `src/services/nativeGhsWithdrawalService.js` reads authority through `isAuthorityEnabled(prisma)` → `isAuthorityModeActive` — also unlocked.

Required correction: claim under the shared lock, in one transaction that also resolves the state, with provider I/O outside it:

```js
await this.prisma.$transaction(async (tx) => {
    await lockAuthorityShared(tx);        // shared, held to commit
    await requireAuthorityState(tx);      // typed UNKNOWN parks the batch
    const claim = await tx.withdrawal.updateMany({ where: { id, status: { in: ['PENDING','APPROVED'] } }, data: { status: 'PROCESSING' } });
    ...
});
```

**Test gap confirmed.** `an UNKNOWN authority state parks the payout batch with zero dispatch` injects `$queryRawUnsafe: async () => { throw ... }`. That proves failure handling and zero dispatch; it does **not** exercise activation racing the claim. `race A` and `race B` exercise the module's lock helpers (`:398`, `:416`), not the worker. A third race is required: worker holds shared across the claim, activation's exclusive must block until commit; and the inverse, activation commits first and the claim must observe `ENFORCED_ACTIVE`.

## 3. Verified this pass

- **Typed 503 propagation:** `src/services/nativeGhsWithdrawalService.js` now calls `fiatLiquidity.isAuthorityEnabled(prisma)` with a comment explicitly refusing to collapse the failure into `false`, so `AuthorityStateUnavailable` reaches the caller. Caveat: that read is unlocked (same class as §2).
- **Durability lane:** `__tests__/wp-h-tristate-authority.pg.test.js` is now in the explicit `SUITES` array of `.github/workflows/financial-durability.yml`.
- **Workflow trigger:** `.github/workflows/test.yml` now runs on `branches: [main, 'integration/**']`, so PRs based on the integration branch finally get the full lane.
- **Real-role protections intact:** re-ran the suite; runtime INSERT/UPDATE/DELETE/TRUNCATE denial, trigger tamper denial, NOLOGIN owner, migrator boundary and operator append-only all still pass.

## 4. Remaining before this slice can be called complete

1. Worker claim under the shared lock (§2) plus a real activation-vs-claim race test.
2. The same lock on the native-GHS withdrawal economic transaction.
3. The 63 legacy fixtures still unconverted.
4. A full exact-head pass once the activation function and fixture conversions land.
