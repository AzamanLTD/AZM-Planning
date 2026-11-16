# WP-H fifth pass — adversarial activation review (PR #400 @ 6ea94867)

Live head **6ea94867b2373ff74d8b1b58f442c00b6b79f768**, tree **4af14802d60d26540acfbf6bd7209e2090a34dcb**, base 6a6f233. Two commits since ec3b3b34 (f9f850e2, 6ea94867). Disposable PostgreSQL 15.19, real distinct LOGINs (`azm_authority_operator` LOGIN, owner/fn_owner NOLOGIN). No `SET ROLE`.

## Executed attacks

| # | Attack | Observed | Status |
|---|---|---|---|
| 1 | Caller-narrowed preflight policy | Signature is now 5 args; old 7-arg call → `function ... does not exist`. Policy is the hard-coded `c_blocking` constant. | FIXED |
| 2 | Substitute each immutability trigger with a same-named no-op | `AUTHORITY_ENFORCEMENT_NOT_ESTABLISHED` / "invokes a substituted function"; absent trigger → "is absent"; disabled → "is not enabled". Record count 0, latch NULL after each. | SAFE |
| 2b | Replace the **body** of `azm_authority_activation_immutable` (same oid, name, owner) | Enforcement did NOT detect it; activation passed enforcement and stopped only at `AUTHORITY_ALREADY_ACTIVATED`. `UPDATE ... SET "evidenceReference"='TAMPERED'` then **succeeded**. | FINDING (High, privileged principal) |
| 3 | Remove each singleton row | `AUTHORITY_SINGLETON_MISSING` naming the exact table; nothing persisted. | FIXED (was my prior Medium) |
| 4 | Pre-poison fn owner: `GRANT ALL ON ALL TABLES` + membership in runtime | Installer exit 0, revoked the membership, **left all 1554 table privileges**. | FINDING (Medium) |
| 5 | PUBLIC EXECUTE window during provisioning | CREATE/OWNER/REVOKE/GRANT are inside one `DO $ddl$` block = one transaction. | SAFE |
| 6 | Missing prerequisite table | Roles installer exit 1 (`42P07` on a leftover constraint name after table rename); function installer exit 1. Not tested on a truly fresh DB without the table. | PARTIAL |

## Analysis

**2b.** PostgreSQL cannot verify a function body's semantics; the function checks trigger identity (`tgfoid`), owner and enabled state, which is the strongest available check. The residual is that anyone able to `CREATE OR REPLACE` that function (fn owner, table owner, superuser) can silently neuter append-only protection while every check still passes. Compensating control: periodic out-of-band attestation of `prosrc`/`pg_proc` against a known digest, plus change alerting.

**4.** Least privilege is proven only for a freshly created role. On an upgraded deployment carrying older, broader grants, the installer grants the minimum on top and never revokes the excess. Fix: `REVOKE ALL ON ALL TABLES IN SCHEMA public FROM <fn_owner>` followed by the minimum grants, and extend the existing catalog validation to fail on any unexpected table privilege.

## Payout worker (unchanged, still High)

`workers/payoutBatchWorker.js` at this head has **0** occurrences of `lockAuthorityShared`/`withAuthoritySnapshot`; the `PENDING`/`APPROVED`→`PROCESSING` claim remains a bare `updateMany`. Both required races remain untested.

## Full-suite failure — `native-ghs-funded-shortfall-reversal`

Assertion `expect(after.usdc).toBe('1000.00000000')` (:404) and the USDC liability check (:413); CI saw 898 then 750 on unchanged code. Locally I ran the file twice on a fresh disposable database: **31/31 both times**. The file contains no `deleteMany`/truncation of ledger or wallet state, and the assertion itself encodes a sound invariant (a correctly reversed funded shortfall restores exactly the USDC that was spent). Verdict: **NOT PROVEN** as an accounting defect; the run-to-run variance points at fixture contamination or suite ordering rather than the reversal logic. Reproduction: run the file in isolation vs inside the CI ordering on the same database and compare the balance delta; if isolation passes and in-order fails, the fix is fixture hygiene, not the assertion.

## Limitations

Not run by me: the operator-activation suite and the full battery; CI logs for run 43272017830/43260435995 were not fetched; the fresh-DB variant of attack 6.
