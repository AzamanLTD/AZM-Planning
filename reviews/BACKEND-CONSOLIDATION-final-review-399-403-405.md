# Backend consolidation — final independent review (#399, #403, #405)

Reviewed 2026-11-16 by Agent 2, read-only. Integrated candidate built locally as #405 merged onto
#403's head (0 conflicts) and exercised on disposable PostgreSQL 15.19.

## Identity

| PR | Head | Tree/branch base | CI at exact head |
| --- | --- | --- | --- |
| #399 | 6a6f2330 | base main 0edefa1 | Test Suite + durability, success, attempt 1 |
| #403 | 157a62b7 | base feature/wp-h-tristate-authority ee3223ae | both lanes success, attempt 1 |
| #405 | c1d84f94 | base wp-i/on-wph 58cbc38 (28 commits behind #403 head) | both lanes + Redis live proof, attempt 1 |

#405 was never CI-tested on top of #403's head. I integrated it locally and ran the new
suites there: 17 suites / 159 tests pass. Release chain on a clean database: 28 installers, 0 failures.

## Findings by area

### 1. WP-H authority — DEFECT (high)
Mutation matrix at the integrated head, real runtime login, 14 mutations:
caught (UNKNOWN): runtime membership in table owner or function owner, SECURITY INVOKER,
function owner change, function owner LOGIN, search_path unpinned, operator INSERT, operator UPDATE,
runtime EXECUTE, PUBLIC EXECUTE.
**Not caught — gate stays `ENFORCED_INACTIVE`:**
- table owner made a LOGIN role (`FiatLiquidityAuthorityActivation` owner);
- trigger-function owner made a LOGIN role;
- trigger-function owner differing from the table owner.
Financial consequence: a positive authority state survives a broken immutability boundary; a
LOGIN owner can drop the append-only triggers and rewrite activation history.
Needed: `ACTIVATION_TABLE_OWNER_UNSAFE`, `IMMUTABILITY_TRIGGER_OWNER_UNSAFE`,
`TRIGGER_OWNER_NOT_TABLE_OWNER` checks plus a pg test that mutates each and asserts UNKNOWN.

### 2. Action-bound step-up — VERIFIED SAFE (with one caveat)
Canonical action (sorted keys, exact decimals, single digest), grant consumed by conditional
`consumedAt IS NULL` UPDATE at `controllers/walletController.js:106` **inside** the economic
`$transaction`, before the balance CAS debit → rollback releases the grant, concurrency has one winner.
Caveat: enforcement defaults OFF (`STEP_UP_ENFORCEMENT`); route falls back to require2FA. Two suites
exercise it ON and pass.

### 3. WP-I provider trust — NOT PROVEN (scope)
Witness service holds its own credentials and its own DB identity; provenance, installer and
reconciler suites pass (7 suites). Net-vs-gross settlement semantics and raw-response integrity
were not inspected line by line in this pass.

### 4. Redis — VERIFIED SAFE
One shared connection layer; production refuses plaintext `redis://` unless explicitly overridden;
scheduler fail-closed when `REDIS_URL` is unset (`SCHEDULER_SINGLE_INSTANCE` must be true);
money-path limiters fail closed, auth/general fail open.

### 5. Release chain — VERIFIED SAFE
New structures are created by committed raw-SQL installers in dependency order
(business-os → wp-i-settlement-witness → business-price → offered-categories → business-currency →
action-step-up-grant → verify-authority-boundary). Verified by running the chain on a clean DB.

### 6. GHS-first business/admin — VERIFIED SAFE (limited scope)
Currency contract and revenue-by-currency suites pass; no combined cross-currency sums found in the
reviewed services.

### 7. Test quality — DEFECT (low)
`__tests__/wp-h-step-up-withdraw-mount.test.js:29` asserts source text
(`expect(src).toContain('isStepUpEnforced() ? next() : require2FA()(req, res, next)')`) — false-positive
prone; replace with a behavioural mount test. DB-gated suites use `describe.skip` without
`TEST_DATABASE_URL`; the durability lane sets it, the general lane must too or the coverage is silent.

## Verdicts

- **#399 — NOT SAFE to merge alone.** WP-E/WP-G/WP-F verified safe by prior executed tests, but the
  authority live decision is still the mutable legacy latch at this head; merging without the
  tri-state wiring leaves financial authority unproven.
- **#403 — REQUIRES CORRECTION before merge.** The WP-H attestation gap above is a high-severity
  false-assurance defect. Everything else in the slice reads correct and its suites pass.
- **#405 — SAFE to merge onto #403 once #403's defect is fixed.** Clean integration, passing currency
  suites, correct release ordering.

Not executed this pass: end-to-end worker race with a live withdrawal row (claim path now takes the
shared lock, re-reads authority inside the transaction and dispatches provider I/O outside — source
verified, lock semantics proven earlier); full battery on the integrated candidate; WP-I deep read.
