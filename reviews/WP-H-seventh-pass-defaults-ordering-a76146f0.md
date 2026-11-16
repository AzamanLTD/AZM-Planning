# WP-H seventh pass — poisoned defaults, installer ordering, twin role identity

**Head** `a76146f0f2447068ebb2bcead999d6cfcac7f105`, **tree** `a28a1ed819a14a84a2ba40c5e6a5d224e8996234`, base `6a6f233`. PostgreSQL 15.19, disposable databases, real LOGINs (`azm_app_runtime`, `azm_authority_operator`), no `SET ROLE`.

## 1. r38 first pass — FIXED (executed)

Clean database, **zero** `azm%` roles cluster-wide, sorted `install-*.js` loop identical to `r38-overlay-upgrade-path`.

Result: **30 installers, all exit 0, first pass, no warming run.** `install-authority-activation-function.js` still sorts before `install-authority-activation-roles.js`, but now short-circuits:

```
"bare overlay/restore invocation: function provisioning and grants not attempted"
exit 0, ok: true
```

Post-sequence roles are correct (`azm_authority_owner`/`_fn` NOLOGIN, runtime/operator LOGIN). Note: the sorted sequence therefore provisions **no** activation function (`pg_proc` count 0) — correct by design, since privileged provisioning must stay out of the release chain, but `ok: true` while skipping can mislead an operator, and no doc points at the explicit step.

## 2 & 3 — HIGH: the audited owner is not the owner holding the function

**Twin role names.**
- `infra/install-authority-activation-roles.js:234` — `` functionOwner: … || `${owner}_fn` `` → **`azm_authority_owner_fn`**
- `infra/install-authority-activation-function.js:25` — `|| 'azm_authority_fn_owner'` → **`azm_authority_fn_owner`**

Catalog proves the divergence: `azm_authority_activate → azm_authority_owner_fn`, while every audit, normalisation and grant in the installer targets `azm_authority_fn_owner`. The audit therefore certifies a decoy principal.

**Measured consequences**

| Attack | Observed |
| --- | --- |
| Poison defaults on the real owner (global + schema tables/sequences → `azm_other`) | 7 `pg_default_acl` rows before **and after**; `ok: true`, `problems: []`, exit 0 |
| Stray relation owned by the real owner (`azm_stray2`) | Not flagged; `ok: true`, exit 0; function OID unchanged (41442459); operator retains `X` |

`infra/lib/authority-default-acl.js` **is** wired (`authority-role-normalisation.js:61`) — it simply runs against the wrong role. Setting `AUTHORITY_FN_OWNER_ROLE` identically for both installers removes the divergence; the defect bites the default/CI path.

## 5. Trigger attestation — confirmed effective (executed)

Body replaced, OID 41442454, name, owner and all trigger references preserved → runtime login returns `UNKNOWN` + `TRIGGER_IMPLEMENTATION_DRIFT`; operator activation refused **before any write** ("the trigger function body has drifted", record 0); restore → `ENFORCED_INACTIVE` → activation → `ENFORCED_ACTIVE`. Trust root is `azm_authority_owner` plus the shipped migration text; byte-exact comparison fails closed on reformatting.

## 6. CI and the worker race

Both lanes green at this head: durability `43463160383`, test suite `43463160117` (4740 passed, 1 skipped). `r38-overlay-upgrade-path` shows PASS, but the workflow retries once on transient failure (`::warning::Test suite failed on first attempt`), so first-attempt-only success is **not** proven by the log.

**Worker race still open (High):** zero `lockAuthorityShared`/`withAuthoritySnapshot` in `workers/payoutBatchWorker.js`; bare `updateMany` claims at `:550` and `:889`.

## 4. Not executed

The p5e and native-GHS-reversal isolation runs (poisoned shared DB, `test:seed:*` keys, double run) were **not** performed this pass. Their green CI status is not, by itself, evidence of isolation.
