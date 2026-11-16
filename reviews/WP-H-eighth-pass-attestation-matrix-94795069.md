# WP-H eighth pass — attestation matrix, default-ACL crash, role resolver

**Live head:** `94795069f7cc95c7eb678e6846b7a9ce96924255` (tree `7d1ff4a1181e3e3e93b64ad1f029b7f4b4c06bed`), reviewed start head `a76146f0`.
**Environment:** disposable PostgreSQL 15.19; real logins `azm_authority_operator`, `azm_app_runtime`, `azm_authority_owner_fn` (temporarily LOGIN for the object-creation proof); no `SET ROLE`.
**CI:** Test Suite `43472706758` and financial-durability `43472706777`, both `run_attempt 1`, both success.

## 1. Role divergence — reproduced, then fixed

At `a76146f` the mixed invocation (roles installer with explicit options, then standalone function installer) gave:

```
proowner = azm_authority_owner_fn        (created by roles installer, ${owner}_fn)
installer log: "enforce function owner azm_authority_fn_owner"   (hardcoded default)
both roles present in pg_roles
```

So the seventh-pass finding was accurate: the audited principal was not the owning principal.

At `94795069`, with every `azm%` role dropped first, the same invocation created only `azm_authority_owner_fn`, `proowner = azm_authority_owner_fn`, and the installer normalised that same role. `infra/lib/authority-roles.js` gives both entry points the single `${owner}_fn` fallback.

**Medium:** `roleScoped` derives only from programmatic options (`install-authority-activation-function.js:50-52`), and the CLI parses no argv. `--function-owner X` is silently ignored; the run reports `ok:true / SKIPPED_NOT_PROVISIONED`.

## 2. Ownership refusal — works in scoped mode

Programmatic scoped call with a stray `public.azm_stray4` owned by the real function owner:

```
ok:false  status: REFUSED_UNEXPECTED_OWNERSHIP
problem: azm_authority_owner_fn owns an unexpected relation: public.azm_stray4
applied: revoked EXECUTE on the existing activation function from PUBLIC and the operator
after: has_function_privilege(operator,…,EXECUTE)=false  has_function_privilege(public,…,EXECUTE)=false
```

Function not replaced. This meets the stated requirement.

## 3. Runtime attestation — 10 of 11 mutations survive as ENFORCED_INACTIVE (**High**)

Baseline `ENFORCED_INACTIVE | USDC | no problems`. Each mutation applied alone, probed as the runtime login, then reverted:

| # | Mutation | Runtime gate |
|---|---|---|
| M1 | activation table owner → LOGIN role | **ENFORCED_INACTIVE** |
| M2 | trigger-function owner → LOGIN role | **ENFORCED_INACTIVE** |
| M3 | trigger-function owner ≠ table owner | **ENFORCED_INACTIVE** |
| M4 | runtime member of table owner | UNKNOWN `RUNTIME_CAN_SET_ROLE_TO_OWNER` ✔ |
| M5 | runtime member of activation-function owner | **ENFORCED_INACTIVE** |
| M6 | activation function → SECURITY INVOKER | **ENFORCED_INACTIVE** |
| M7 | activation-function owner changed | **ENFORCED_INACTIVE** |
| M8 | activation-function owner → LOGIN | **ENFORCED_INACTIVE** |
| M9 | activation-function search_path changed | **ENFORCED_INACTIVE** |
| M10 | operator granted direct INSERT on the record | **ENFORCED_INACTIVE** |
| M11 | runtime / PUBLIC granted EXECUTE on the activation function | **ENFORCED_INACTIVE** |

The gate inspects the table, its triggers and trigger body, and runtime membership in the **table** owner. It does not inspect the activation function at all, nor LOGIN attributes, nor membership in the function owner, nor operator privileges on the record.

## 4. Default-ACL normaliser aborts on PG 15 (**High**)

Poisoned defaults for the real function owner (global + schema, tables/sequences/functions/types, `azm_other` and PUBLIC), then the scoped installer:

```
rows before 5 → after 4   (global TABLES revoked, the rest left)
PrismaClientKnownRequestError P2010
message: ERROR: role "unknown (OID=0)" does not exist
```

`pg_get_userbyid(0)` on PG 15 returns the pseudonym `unknown (OID=0)` rather than NULL, so the PUBLIC branch in `revokeDefaultPrivileges` is unreachable and the run aborts mid-normalisation.

Consequence proven with a real login as the function owner:

```
CREATE TABLE scratch.t_future  →
t_future ACL: =arwdDxt/azm_authority_owner_fn | azm_authority_owner_fn=arwdDxt/…
has_table_privilege('azm_other','scratch.t_future','SELECT') = true
```

PUBLIC receives ALL and the unrelated role receives SELECT on objects created after installation.

## 5. Worker claim race — unchanged (**High**)

`workers/payoutBatchWorker.js` at the live head: zero `lockAuthorityShared`/`withAuthoritySnapshot` calls. Authority predicate read unlocked at `:259` and `:319`; bare `updateMany` claim at `:550` (`{id, status in ['PENDING','APPROVED']}` → `PROCESSING`); manual-review claim at `:889` (`status in claimableStatuses` → `NEEDS_MANUAL_REVIEW`), also bare. Activation can commit between the predicate read and the claim, and the claim then commits under the pre-activation snapshot.

## 6. Not executed

The p5e and native-GHS isolation experiments. No claim is made about them.
