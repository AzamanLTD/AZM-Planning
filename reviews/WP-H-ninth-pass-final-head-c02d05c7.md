# WP-H ninth pass — final-head verification at c02d05c7

**Exact head:** `c02d05c727c77f32f6ab755b914dae38317e548e` · **tree** `127d983d1cf1363af07982ee5b4f94f870e126b1`
**Base:** `integration/main-reconciliation` · PR #400 open, draft
**PostgreSQL:** 15.19 · real LOGINs (`azm_authority_operator`, `azm_app_runtime`), no `SET ROLE`
**CI at this head:** `43527560905` financial-durability success (attempt 1), `43527560997` Azaman Test Suite success (attempt 1)

## Verdict

| Item | Result |
| --- | --- |
| P1 ownership refusal | **PASS** |
| P2 single resolver | **PASS** |
| P2 CLI `--function-owner` | **FAIL — silently inert** |
| P3 attestation matrix | **FAIL — 12 of 14 mutations keep a positive state** |
| P4 disposable cleanup | **PASS** |
| P5 default-ACL normaliser | **PASS** |
| P6 worker claim races | **FAIL — mechanism absent** |

## P1 — PASS
Stray relation owned by the real function owner → `{"ok":false,"status":"REFUSED_UNEXPECTED_OWNERSHIP","problems":["azm_authority_owner_fn owns an unexpected relation: public.azm_stray"]}` with `revoked EXECUTE on the existing activation function from PUBLIC and the operator`. Measured after: `has_function_privilege(operator,…)=false`, `has_function_privilege(public,…)=false`. Function not replaced.

## P2 — resolver PASS, CLI FAIL
`infra/lib/authority-roles.js` resolves `${owner}_fn` for every entry point the roles installer drives; the function is owned by `azm_authority_owner_fn` and the same principal is audited.

The standalone CLI has **no argv parsing** (`roleScoped` derives only from programmatic options, `infra/install-authority-activation-function.js:50-52`). Executed:

```
node infra/install-authority-activation-function.js --function-owner azm_custom_fn
→ EXIT=0, role azm_custom_fn never created, proowner unchanged (azm_authority_owner_fn), no warning
```

An operator passing the documented flag gets a silent no-op and exit 0.

## P3 — FAIL (HIGH)
Baseline as the runtime login: `ENFORCED_INACTIVE | USDC`. One mutation at a time, reverted after each:

| Mutation | Runtime result |
| --- | --- |
| table owner → LOGIN role | ENFORCED_INACTIVE ✗ |
| trigger-function owner → LOGIN role | ENFORCED_INACTIVE ✗ |
| trigger owner ≠ table owner | ENFORCED_INACTIVE ✗ |
| runtime member of table owner | UNKNOWN (RUNTIME_CAN_SET_ROLE_TO_OWNER) ✓ |
| runtime member of trigger owner | UNKNOWN ✓ |
| runtime member of activation-function owner | ENFORCED_INACTIVE ✗ |
| activation fn SECURITY INVOKER | ENFORCED_INACTIVE ✗ |
| activation fn owner change | ENFORCED_INACTIVE ✗ |
| activation fn owner → LOGIN | ENFORCED_INACTIVE ✗ |
| activation fn search_path change | ENFORCED_INACTIVE ✗ |
| operator INSERT on record | ENFORCED_INACTIVE ✗ |
| operator UPDATE on record | ENFORCED_INACTIVE ✗ |
| runtime EXECUTE on activation fn | ENFORCED_INACTIVE ✗ |
| PUBLIC EXECUTE on activation fn | ENFORCED_INACTIVE ✗ |

The gate never inspects the activation function, LOGIN attributes on either owner, operator DML on the record, or membership in the function owner. Highest impact: `PUBLIC EXECUTE` and `operator INSERT` both leave the gate green while the sanctioned path is fully bypassed.

## P5 — PASS
Poisoned six default-privilege entries (global and `IN SCHEMA scratch`; tables, sequences, functions, types; to `azm_other` and PUBLIC). After the normaliser: 2 rows remain, both owner-only (`azm_authority_owner_fn=U` on types, `=X` on functions); `ok:true`, `PROVISIONED`, `problems:[]`. Objects created afterwards as the function owner (real login): `relacl NULL` (owner-only), `proacl` owner-only, `azm_other` SELECT=false INSERT=false sequence-USAGE=false, PUBLIC grantee entries 0. The PG15 `pg_get_userbyid(0)` crash is fixed by the `CASE WHEN a.grantee = 0::oid` branch in `infra/lib/authority-default-acl.js`.

## P4 — PASS
From a cluster with zero `azm%` roles: p5e `15 passed` twice, after which **no `azm_p5e*` database and no `azm%` role** remained (the suite drops `migrator/owner/runtime/operator/owner_fn`). Shortfall suite `31 passed` twice, no leftover database. Forced setup failure (child_process stubbed to throw): helper threw `forced setup failure` and left **no** `azm_forced*` database.

## P6 — FAIL (HIGH)
`workers/payoutBatchWorker.js` at this head contains **0** occurrences of `lockAuthorityShared`/`withAuthoritySnapshot`; the claim is still a bare `withdrawal.updateMany` outside any transaction.

Two-session evidence:
- claim-shaped transaction open, no authority lock → activation returned `true` in **0 s** (no serialization);
- identical run with a holder of `pg_advisory_xact_lock_shared` → activation blocked to the **2 s** statement timeout.

The lock primitive works; the worker does not use it. Worker-first and activation-first orderings, UNKNOWN, CAS loss and timeout recovery therefore cannot be demonstrated as passing — they are unimplemented.

## Limitations
CI gate results are from the run list only; I did not parse per-suite counts, skip reasons or retry warnings for these two run IDs. P6 was exercised at the lock level, not through a live worker process. No backend source was edited; all databases, roles and worktrees used here are disposable and were removed.
