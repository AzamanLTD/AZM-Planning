# WP-H review artifact — candidate 6a6f233 (Agent 2, read-only)

Source: independent review of `AzamanLTD/AZM-backend` PR #399 at `6a6f23304c7a2fdfe8e3e0a60b61dda97e07ebd3`. GitHub blocks a formal REQUEST_CHANGES review on a PR authored by the same account (HTTP 422 "Can not request changes on your own pull request"), so the report is published here verbatim.

## Independent review — candidate `6a6f23304c7a2fdfe8e3e0a60b61dda97e07ebd3`

Read-only review by Agent 2. Reviewed tree: `6a6f23304c7a2fdfe8e3e0a60b61dda97e07ebd3` (base `0edefa1dbe056343080f2d762d89a1b0936c109c`, 186 commits ahead, 236 changed files). WP-H patch reviewed unapplied: `docs/wp-h/authority-live-decision.patch`, sha256 `1bb0598fd1bb7f001bdd586b1018654b13cc9c41330f8efffd810ddec4743787`.

### Verified by execution (disposable PostgreSQL 15.19, database built from this candidate's own 24-step release sequence)

- **WP-G — PASS.** `jest __tests__/route-specific-fx-authorization-gate.test.js --runInBand --forceExit` → 19 passed / 19 total, exit 0. Ordering read from the router source, not comments: `routes/walletRoutes.js` mounts `protectActive → requireKyc('withdraw') → require2FA() → requireFreshFxRate({ when: 'MOMO_PAYOUT_DESTINATION' }) → idempotency(...)`. Negative KYC cases (PENDING/REJECTED/UNVERIFIED) still refused by KYC with KYC's own code; KYC enforcement was not weakened to obtain the count.
- **WP-E real-role suite — PASS.** `jest __tests__/p5e-enforced-authority-activation.pg.test.js --runInBand --forceExit` → 15 passed / 15 total. Distinct migrator / NOLOGIN owner / restricted runtime / operator identities, not superuser + `SET ROLE`. Runtime can SELECT the activation record; INSERT, UPDATE, DELETE and TRUNCATE are refused; triggers cannot be altered, dropped or disabled; `SET ROLE` to owner/operator is refused.

### Verified by source (not executed by me)

- **T9 / release contract — PASS on source.** Commit `6a6f233` changes only `__tests__/transit-booking-identity.pg.test.js`: the fake response now settles once behind a `settled` guard and captures the claim at `res.json`. No production code in that commit. The contract in `middleware/idempotency.js` is conditional and ordered: `deleteMany({ where: { id: claim.id, status: OPERATION_IN_PROGRESS } })` (:351-353) means a concurrent `COMMITTED` transition is a no-op, never a deletion; `releaseClaim().finally(() => originalJson(body))` (:406) means release completes before the response is delivered; 4xx release requires explicit `res.locals.financialClaimRelease` or route-level `releaseOn4xx`; transient errors retry bounded, non-transient or exhausted release leaves IN_PROGRESS so the key stays poisoned. I did not execute the T9 suite end to end.
- **Dependency audit** is a separate required job (`.github/workflows/test.yml:426-432`) retaining `--audit-level=low`.

## Requesting changes — WP-H (`docs/wp-h/authority-live-decision.patch`)

The patch closes the real bypass (the runtime role can no longer flip the regime by writing `FiatLiquidityState.authorityActivatedAt`), and that direction is correct. Five blockers remain before it can land.

**1. Enforcement failure must not become `active:false`.** `src/services/authorityActivationEnforcement.js:228+` returns `enforcedActive: false` for every unavailable condition — `TABLE_MISSING`, `ENFORCEMENT_UNAVAILABLE`, `ACTIVATION_UNREADABLE` — and the patch maps `enforcedActive === true` to `active`, so all of them resolve to `authorityUnit: 'USDC'` with `projectionBalance`. A deployment whose activation table is missing, whose roles were never provisioned, whose owner is wrong, whose triggers are disabled, or whose record is unreadable would present live GHS liabilities as USDC and book USDC against GHS balances.

**2. `UNKNOWN` must become a typed refusal.** `readMode`/`readModeAndPool` must be tri-state: `ENFORCED_ACTIVE` only when `enforcedActive === true`; `ENFORCED_INACTIVE` only when `reason === NOT_ACTIVATED` **and** `enforcementEstablished === true`; every other outcome `UNKNOWN`, which must raise `AUTHORITY_STATE_UNAVAILABLE` from the money paths. The discriminator already exists in `reason`; the patch keeps it only in diagnostics and discards it for the decision.

**3. Only positively verified states may be acted on.** No default selection of USDC or GHS on an unverified state, and no consumer may infer the regime from a Boolean.

**4. A missing activation row needs advisory-lock serialization.** No row lock can cover a row that does not exist, so `SELECT ... FOR SHARE` on the activation record is insufficient. Take `pg_advisory_xact_lock(ACTIVATION_LOCK_KEY)` in **shared** mode on the money path and **exclusive** in activation, so an activation commit cannot interleave between the regime read and the balance it labels. `lockRegime` exists today but is used only by the activation sequence, not by `readMode`.

**5. Money operations must hold the shared lock through the economic write.** The lock must span read → economic write → commit, not just the read, otherwise a withdrawal can read INACTIVE, activation commits, and the withdrawal books USDC under a GHS regime.

**6. The 63 tests must use operator-backed activation and runtime execution.** They currently simulate the GHS regime by writing the legacy flag or latch (e.g. `__tests__/p5d-fiat-liquidity-authority.test.js:161-164` calls its reset "a TEST-ONLY reset of the one-way [latch]"; `__tests__/p5d-latched-regime-contract.test.js:58-59,88` seeds through the same column). They must provision real separated roles as `p5e` does, activate by inserting the record as the **operator**, and run the financial operations as the **runtime** identity, preserving every economic assertion. Converting them by granting the fixture extra privileges, or by restoring a column write, would re-open the bypass and invalidate the count.

### Not reviewed here
WP-I provider-event provenance is unresolved and separately reported. I did not re-run the full 4,682-test battery, the durability lane, or the WP-F concurrency suites; those remain Agent 1's evidence, not mine.

### Verdict
- **WP-H design: REQUIRES CORRECTION** (`REQUEST_CHANGES`).
- **Candidate integration readiness: NOT READY** until WP-H lands with a tri-state authority model and the serialized no-record path.
