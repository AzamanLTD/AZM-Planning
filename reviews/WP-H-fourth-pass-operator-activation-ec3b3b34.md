# WP-H fourth pass — operator-only activation function (PR #400 @ `ec3b3b34c1e20947a416089ab1852213b5c68036`)

Read-only review by Agent 2. Tree `d25f7c70f2580c0cb734759cb9e4ed75b65e517f`, base `integration/main-reconciliation` at `6a6f23304c7a2fdfe8e3e0a60b61dda97e07ebd3`. Head moved twice during review (requested `94abb316` → `c703e14f` → `ec3b3b34`); this section reviews the live head only.

## Verdict: the activation function is sound. Two findings, both bounded. Payout-worker race still open.

## Executed (disposable PostgreSQL 15.19; real distinct LOGINs; catalog inspected directly; no `SET ROLE` as proof)

Provisioned with the repo's own installers (`install-authority-activation-roles`, `install-authority-activation-function`) against a database built from the candidate's release sequence.

### 1–7. Privilege boundary — all twelve required properties verified

| Check | Evidence |
| --- | --- |
| Function owner dedicated NOLOGIN, not the table owner | `pg_roles`: `azm_authority_fn_owner` rolcanlogin=f, rolsuper=f, rolinherit=f; table owner is `azm_authority_owner` |
| Operator cannot INSERT directly | `psql -U azm_authority_operator` → `ERROR: permission denied for table FiatLiquidityAuthorityActivation` (INSERT, UPDATE, DELETE all denied) |
| Runtime SELECT-only, cannot assume either role | `SET ROLE azm_authority_operator` / `azm_authority_fn_owner` → `ERROR: permission denied to set role` |
| EXECUTE ACL limited to the operator | `proacl = {azm_authority_fn_owner=X/..., azm_authority_operator=X/...}` — no PUBLIC entry |
| SECURITY DEFINER, pinned search path, schema-qualified | `prosecdef=t`, `proconfig={search_path=pg_catalog}`; every object `public.*`/`pg_catalog.*` |
| Least-privilege function owner | table-level: `INSERT,SELECT` on the record, `INSERT` on audit, `SELECT` on five read tables; **column-level UPDATE only**: `FiatLiquidityState.authorityActivatedAt`, `GlobalSettings.fiatLiquidityAuthorityEnabled`, `SystemFiatPool.balance` — no table-level UPDATE anywhere |
| No membership abuse | `pg_auth_members` empty for all `azm%` roles |
| `session_user` authorization | function compares `session_user` (not `current_user`), so a definer switch cannot satisfy it; `p_operator <> session_user` rejected |

### 8–11. Atomicity, rollback, concurrency, serialization

- **Late-stage rollback proven.** Revoked the audit INSERT from the function owner so step 11 fails *after* the record, latch, flag and projection writes had executed. Result: `ERROR: permission denied for table AdminSettingsAuditLog`, and the database showed record count 0, latch NULL, flag unset, projection untouched. One function = one transaction; nothing partial survived.
- **Concurrent activations: exactly one winner.** Two operator calls launched 200 ms apart → `A` succeeded, `B` raised `AUTHORITY_ALREADY_ACTIVATED`; final state one record, one audit row.
- **Serialization with economic readers.** Held `pg_advisory_xact_lock_shared(hashtext('azm.p5d.authority.activation'))` in a second session; the activation call blocked for the full 2 s `statement_timeout` instead of proceeding. Same key, shared/exclusive, transaction-scoped.

### 12. No legacy-only partial activation from the app route

`src/services/fiatLiquidityActivationService.js` now throws `AuthorityActivationSanctionedProcedureRequired` (`AUTHORITY_ACTIVATION_SANCTIONED_PROCEDURE_REQUIRED`, 409, "No partial latch was written") so the ordinary app activation path cannot set the latch without the protected record.

## Finding 1 — Medium — the function does not verify its latch/flag/projection writes landed

Steps 9–10 are `UPDATE ... WHERE "id" = 1` with no row-count check. In a database where `GlobalSettings` and `SystemFiatPool` rows do not exist (verified: both tables had **0 rows** in my build), the function returned success while those two writes affected **0 rows**: record inserted, latch set, flag never set, projection never synced. The protected record is authoritative so the regime is still correct, and the projection is inert under tri-state — but under the default `legacy` mode `readModeAndPool` still reads those fields.

Correction: `GET DIAGNOSTICS v_rows = ROW_COUNT;` after each update and `RAISE` unless `v_rows = 1`.
Regression test: activate against a database with the singleton rows absent and assert a typed refusal (and no record).

## Finding 2 — High — payout worker claim race still open at this head

`workers/payoutBatchWorker.js` is untouched by these three commits (0 changed lines). The claim remains:

    const claim = await this.prisma.withdrawal.updateMany({
        where: { id: withdrawal.id, status: { in: ['PENDING', 'APPROVED'] } },
        data: { status: 'PROCESSING' }
    });

Bare client, no transaction, no shared lock, and `grep` finds no `lockAuthorityShared`/`withAuthoritySnapshot` in the worker. The `_processBatch()` UNKNOWN guard is a separate unlocked read, so activation can commit between the guard and the claim. The `zero dispatch` test proves failure handling, not serialization.

Required: resolve authority and claim the row inside one `$transaction` after `lockAuthorityShared(tx)`, release before provider network I/O, and add both races (claim-first, activation-first).

## Not verified
Full-suite/CI execution of the new `wp-h-operator-activation.pg.test.js`; the worker-lock implementation (not yet written); reversal/MoMo consumer wiring; production role provisioning against Render's single-credential deployment.
