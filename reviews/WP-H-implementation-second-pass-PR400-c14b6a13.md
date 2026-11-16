# WP-H second-pass review — PR #400 `c14b6a13` (Agent 2, read-only)

**Review object (verified):** `AzamanLTD/AZM-backend` PR #400, branch `feature/wp-h-tristate-authority`,
head `c14b6a1310a6389de94746f23f5a2e22d7a2267e`, tree `f0a8fa19aca69cc907e6bb31685758c39b27a899`,
base `integration/main-reconciliation` @ `6a6f23304c7a2fdfe8e3e0a60b61dda97e07ebd3` (merge-base equals base, 1 commit, 4 files, +773).
Earlier review (candidate 6a6f233) is preserved in `WP-H-authority-live-decision-review-6a6f233.md`.

## Verdict: READY FOR CONSUMER WIRING

The primitives are correct, positive-only and dormant. The boundary is **not** yet in force, and three
wiring defects must be fixed before `AUTHORITY_ENFORCEMENT_MODE=tri_state` is ever set.

## Executed (not merely inspected)

- Disposable PostgreSQL 15.19 at the PR head: `prisma db push` + the candidate's own release-installer
  sequence + `prisma generate`.
- `__tests__/wp-h-tristate-authority.pg.test.js` → **15 passed / 15**, exit 0. The suite creates four
  real logins (`CREATE ROLE … NOLOGIN` owner, LOGIN runtime/operator/migrator) and connects per-URL —
  no `SET ROLE` simulation.
- Advisory-lock semantics probed directly through the module's own helpers, three separate
  transactions: shared acquired → exclusive refused (`false`) → shared+shared allowed (`true`) →
  after commit exclusive available (`true`) → while exclusive held, shared refused (`false`).
  So `$executeRawUnsafe('SELECT pg_advisory_xact_lock_shared(hashtext($1))')` does acquire the
  transaction-scoped lock under this Prisma/Postgres, and the single-`int4` `hashtext` → `bigint`
  overload resolves correctly.
- CI: head `c14b6a13` has exactly one run, `financial-durability` `43128404899` (success). That lane
  selects suites from explicit bash arrays (`financial-durability.yml:349,480`) and does **not**
  include the new suite; `test.yml` (default discovery `npm test`, `:388`) produced no run for this
  head. **The new suite has zero CI coverage; my local 15/15 is the only execution evidence.**

## Critical — `readModeAndPool()` is unchanged and publishes a currency

`src/services/fiatLiquidityActivationService.js` `readModeAndPool()` still computes
`active = flagged || latched` from `GlobalSettings.fiatLiquidityAuthorityEnabled` and
`FiatLiquidityState.authorityActivatedAt`, then returns `authorityUnit`/`currency` and a `balance`
(`availableGhs` vs `projectionBalance`). It has **no** tri-state branch.

Production consumers (7): `controllers/adminController.js:964,2184,2322`,
`controllers/adminProfitBreakdownController.js:48`, `controllers/finance.controller.js:585`,
`scripts/p5d-authority-activate.js:78` (+ the module's own reads).

Consequence: with `AUTHORITY_ENFORCEMENT_MODE=tri_state` set, a runtime role that clears the latch
still causes admin pool views, the profit breakdown and the finance summary to report **USDC with the
`SystemFiatPool` figure** for GHS-backed liquidity. That is the exact defect WP-H exists to close,
still live on the reader path. Correction: route `readModeAndPool` through `requireAuthorityState`
and refuse `AUTHORITY_STATE_UNAVAILABLE` under tri-state, or expose the pool figures only with the
resolved unit and no fallback. Regression test: clear the latch as runtime under tri-state and assert
each of those endpoints refuses rather than answering USDC.

## High — the economic lock has no production consumer

`withAuthoritySnapshot(tx, fn)` exists and is correct, but `git grep` shows **zero** production call
sites outside `authorityState.js`. The economic readers use plain `readMode(tx)`:
`services/finance.service.js:174` (withdrawal authority), `:763`, `:848` (reversals), `:889`
(liquidation) — each reads the regime **without** taking the shared lock, so the no-record race the
brief targets is open in the real path even though the protocol itself is proven sound.

Correction: every economic consumer must call `withAuthoritySnapshot(tx, …)` inside the same
transaction that performs the write, and retain it through commit. Regression test: two transactions —
economic path holds shared while activation attempts exclusive (must block), and the reverse.

## High — the two activation writers disagree

`activateAuthority()` (`fiatLiquidityActivationService.js`) takes the exclusive `lockRegime`, writes
`FiatLiquidityState.authorityActivatedAt` and an audit row, and **never inserts the protected
`FiatLiquidityAuthorityActivation` record**. That record is created only by the operator/DBA script.

Under tri-state this means an operator using the app-side activation path sets the legacy latch with no
protected record → resolver returns `UNKNOWN` (`LEGACY_CONTRADICTS_INACTIVE`) → every money path
refuses. Fail-closed and therefore not a money-safety defect, but it makes the app path a trap.
Correction: `activateAuthority` must refuse under tri-state and direct the operator to the privileged
path (it cannot insert the record itself — the runtime role has no INSERT by design), or the two paths
must be unified. Regression test: tri-state + app activation attempt → typed refusal, no latch write.

## Medium — the rollout switch does not cover the reader path

`enforcementMode()` returns `tri_state` only when `AUTHORITY_ENFORCEMENT_MODE === 'tri_state'`,
defaulting to `legacy`. Because `readModeAndPool` is outside the switch, flipping the flag yields a
**partially** switched system. `docs/wp-h/tristate-authority.md` is candid — "Not yet done in this
branch … consumer-by-consumer wiring" — so the branch cannot be mistaken for completed protection,
provided the PR body repeats it and the flag is not set before wiring lands.

## Verified correct

Tri-state semantics (`validateActivation` rejects malformed/singleton/digest violations; legacy
latch-set-with-no-record → `UNKNOWN`; only positively inspected `ENFORCED_INACTIVE` with an agreeing
latch yields USDC); typed `AuthorityStateUnavailable` carrying no `unit`/`active`/`balance` property;
shared/exclusive key identity asserted against `ACTIVATION_LOCK_KEY`; real-login roles and
privilege/trigger/DDL denials (15/15 locally, matching the p5e 15/15 I reproduced at `6a6f233`).

## Missing CI coverage (minimum correction)

Add `__tests__/wp-h-tristate-authority.pg.test.js` to the `SUITES` array in
`.github/workflows/financial-durability.yml` (the lane that owns financial invariants) and confirm the
suite appears by name in the run log. Also confirm the main `test.yml` lane runs for PRs targeting
`integration/main-reconciliation` — it produced no run at this head, so discovery-based coverage that
would otherwise include `*.pg.test.js` never executed.

## The 63 historical failures — unchanged, migration plan

Status: not classified or converted in this branch. Plan per suite, preserving every economic
assertion:

1. `p5d-fiat-liquidity-authority` — fixture writes `authorityActivatedAt` at `:161-164` ("TEST-ONLY
   reset of the one-way latch"). Convert to: provision owner/runtime/operator/migrator logins, install
   the table as migrator, insert the record **as operator**, run the economic assertions **as runtime**.
2. `p5d-latched-regime-contract` — seeds via the column at `:58-59,88`. Same conversion; keep the
   "Boolean cleared after activation changes nothing" assertion, which becomes stronger when the
   record is the source.
3. `p5d-legacy-under-authority-regime` — legacy assertions must run under `ENFORCED_INACTIVE` with a
   verified record absent and a clean legacy latch; add an explicit UNKNOWN case.
4. `native-ghs-withdrawal` — withdrawal economics against GHS needs operator-backed activation and a
   runtime connection; no fixture may gain INSERT/UPDATE on the activation table.

None of the four may be made green by granting the runtime fixture extra privileges, by writing the
latch directly, or by re-adding a legacy fallback.
