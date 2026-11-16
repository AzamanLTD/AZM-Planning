# WP-H sixth pass — trigger attestation, poisoned owner, atomicity, CI

Reviewed head **`42e90aae36da170b6926805f9dc74ff21e2ec5e3`**, tree `8a7cce88a46b2d26d8deac3397ab06e26e4aae75`
(base `6a6f233`). Two commits since the last pass: `943cda04` (verify the trigger's implementation, normalise a
poisoned function owner) and `42e90aae` (an overlay restore is not a role-scoped install).

Environment: disposable PostgreSQL 15.19, distinct real LOGINs (`azm_authority_operator`, `azm_app_runtime`,
`azm_authority_owner`, `azm_authority_fn_owner`), no `SET ROLE`.

## Attack 1 — same-OID trigger body replacement: FIXED, both sides (executed)

Replaced the body of `azm_authority_activation_immutable()` with a no-op while preserving OID (41424854),
name, owner (`azm_authority_owner`) and all three trigger definitions.

- App side, connected as the **runtime login**: `{"state":"UNKNOWN","unit":null,
  "reason":"ENFORCEMENT_NOT_ESTABLISHED","problems":["TRIGGER_IMPLEMENTATION_DRIFT"]}` — no unit emitted.
- SQL side, as the operator: `AUTHORITY_ENFORCEMENT_NOT_ESTABLISHED` / "the trigger function body has drifted
  from the shipped canonical implementation" (raise at line 148), record count 0, latch NULL.
- Restored the canonical body from the shipped migration: runtime inspection returned
  `ENFORCED_INACTIVE` / USDC; activation then succeeded and inspection returned `ENFORCED_ACTIVE` / GHS.

Residual on the fingerprint (source-level): the comparison is byte-exact against `prosrc`, and the expected text
is injected into the SQL function at install time from `src/services/authorityTriggerContract.js`, with the
migration as the declared anchor and a drift test enforcing agreement. No database-stored digest is trusted,
which is the right call — a digest row would be writable by the same principal. Two caveats worth recording:
byte-exact comparison fails closed on harmless migration reflow (safe, but brittle), and the activation function
itself is owned by `azm_authority_fn_owner`, so a principal holding that role's DDL rights can rewrite the
verifier as well as the trigger. The trusted root is therefore "who holds function-owner DDL", which should be
stated in the runbook and monitored via `pg_proc` change detection.

## Attack 2 — poisoned function owner: normalised except two gaps (executed)

Poisoned a pre-existing `azm_authority_fn_owner`: `LOGIN CREATEROLE`, `GRANT ALL ON ALL TABLES/SEQUENCES/ROUTINES`,
membership in `azm_other` (which held `ALL` on all tables), ownership of a stray table `azm_fn_stray`, and
`ALTER DEFAULT PRIVILEGES FOR ROLE azm_authority_fn_owner ... GRANT ALL ON TABLES TO azm_other`.

| Vector | Poisoned | After shipped installer |
| --- | --- | --- |
| `rolcanlogin` | true | **false** |
| `rolcreaterole` | true | **false** |
| membership `azm_other` | present | **revoked** |
| direct table privileges | 1533 | **16** (minimum) |
| owned stray relation | `azm_fn_stray` | **still owned** |
| `pg_default_acl` entry | 1 | **1 — not inspected, not removed** |

The installer does not claim success: it reports `ok:false` with "owns an unexpected relation: public.azm_fn_stray"
and exits 1. That satisfies the "no false success" test, but two things remain.

**Finding M-1 (Medium).** `pg_default_acl` is absent from the installer's catalog verification. The poisoned
`ALTER DEFAULT PRIVILEGES` survives, so any future object created by that role silently grants `ALL` to
`azm_other`. Fix: inspect `pg_default_acl` for the role both as grantor and as grantee and either remove the
entries or fail with a named problem.

**Finding L-1 (Low).** Stray ownership is detected but not remediated, and the function is installed anyway,
leaving a partially normalised role behind an exit-1 install. Fix: `ALTER … OWNER TO` the sanctioned owner or
refuse before installing.

## Attack 3 — atomicity and serialization: PASS (executed)

Independently failed each write stage by revoking the function owner's grant, and asserted every activation
write rolled back each time:

| Injected failure | record / latch / flag / pool / audit afterwards |
| --- | --- |
| `GlobalSettings."fiatLiquidityAuthorityEnabled"` UPDATE | 0 / NULL / false / 0.00 / 0 |
| `SystemFiatPool.balance` UPDATE | 0 / NULL / false / 0.00 / 0 |
| `AdminSettingsAuditLog` INSERT (last stage) | 0 / NULL / false / 0.00 / 0 |

Two concurrent activations 200 ms apart: one returned `activated: true`, the other
`AUTHORITY_ALREADY_ACTIVATED`; final state exactly one record and one audit row. With a second session holding
`pg_advisory_xact_lock_shared` on the regime key, activation blocked for its full 2 s `statement_timeout`.

## Attack 4 — durability lane: PASS, verified from the CI log (not the workflow file alone)

`financial-durability.yml` selects all five WP-H suites (`:479-491`), and the log of job `131601961057`
(run `43338891494`, head `42e90aae`) shows them actually executing in the production-commit-mode lane:

```
PASS __tests__/wp-h-tristate-authority.pg.test.js
PASS __tests__/wp-h-operator-activation.pg.test.js
PASS __tests__/wp-h-authority-policy-drift.test.js
PASS __tests__/wp-h-trigger-integrity.pg.test.js
PASS __tests__/wp-h-function-owner-privileges.pg.test.js
Test Suites: 67 passed, 67 total   Tests: 760 passed, 760 total
```

## B-1 — BLOCKER: the general suite is RED at this head

Run `43338891482` ("Azaman Test Suite", job `test`) concluded **failure** at `42e90aae`:

```
FAIL __tests__/r38-overlay-upgrade-path.test.js
     "UNEXPECTED DDL failure → overlay exits NON-ZERO" — Expected: 0, Received: 1
FAIL __tests__/native-ghs-funded-shortfall-reversal.test.js
FAIL __tests__/p5e-enforced-authority-activation.pg.test.js
Test Suites: 2 failed, 1 skipped, 448 passed (of 450)   Tests: 23 failed, 4717 passed (of 4741)
```

The `r38-overlay-upgrade-path` failure is the most likely to be a direct regression from `42e90aae`, which
touches overlay restore semantics. I did **not** reproduce these three suites locally, so this is CI evidence,
not my execution. The durability lane being green while the general lane is red is precisely the asymmetry the
gate must not tolerate.

## Outstanding from earlier passes

- **Payout worker (High, unchanged):** `workers/payoutBatchWorker.js` is not in this diff; the
  `PENDING`/`APPROVED → PROCESSING` claim is still a bare `updateMany` with no shared-lock transaction.
- **Shortfall-reversal drift (unresolved):** still failing in CI on the same invariant
  (`expect(after.usdc).toBe('1000.00000000')`). It passed 31/31 twice on my fresh database, and the file has no
  cleanup, so fixture/ordering contamination remains the leading hypothesis — still not proven.
