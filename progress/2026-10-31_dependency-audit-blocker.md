# Dependency audit blocker and Firebase replacement state

**Date:** 2026-10-31
**Status:** blocker open, awaiting owner decision. Nothing merged, deployed or wired.

## Why this memo exists

`CURRENT_STATE.md` documents backend main at `6802ab6` (2026-09-15). The October
PR stack (#384, #385, #387, #388) and the dependency-audit blocker are not
recorded anywhere in this repo. This memo captures only measured facts.

## The gate and its current result

`npm audit --omit=dev --audit-level=low` on #384 head `73109bea`: **4 moderate, 0 high.**

```
rimraf          >=4.2.0            node_modules/rimraf
└ glob          3.0.0 - 14.0.1     node_modules/rimraf/node_modules/glob    (AbortSignal bypass)
└ minimatch     <11.0.1            node_modules/rimraf/node_modules/minimatch (GHSA-g3hv-75wg-m249)
google-gax      5.0.5 - 5.0.8      node_modules/google-gax                  (via rimraf)
```

Single origin: `firebase-admin@14.3.0` declares `@google-cloud/firestore: ^8.7.1`
as an **optional** dependency → `google-gax@5.0.8` → `rimraf@5.0.10` →
`glob@10.5.0` → `minimatch@9.0.9`. Newest 8.x is 8.7.1, so no in-range patch exists.

Firestore is not used by production code: no `getFirestore` and no `firestore()`
call exists outside tests.

## Four routes, all measured and all dead

| Attempt | Measured result |
|---|---|
| `npm audit fix` (no `--force`) | **0** lockfile entries changed; `fixAvailable: true` is misleading |
| `firebase-admin@14.5.0` (optional firestore `^9.1.0`) | **6 moderate**, adds `gaxios@7.1.3`, `gcp-metadata@8.1.3-8.1.4`, nests `google-gax@5.0.8` under `@google-cloud/firestore-api` |
| `@google-cloud/firestore@9.4.0` alone | **6 moderate**, same additions |
| `npm ci --omit=optional` | still **4**: audit resolves the lockfile, not the installed tree. A clean tree holding neither `google-gax` nor `rimraf` still reported all four |

Relevant detail: `google-gax@6.14.0` has **no `rimraf` dependency at all**, so the
cluster exists only in the 5.x line, yet every route to 6.x currently adds worse advisories.

Earlier override experiments on this tree degraded it to 12-13 findings including 3 high.
Do not repeat them.

## CI evidence reality

`.github/workflows/test.yml` triggers only on pushes to `main`/`develop`,
pull requests whose **base is `main`**, `merge_group`, and manual dispatch.
`financial-durability.yml` triggers on every push.

Consequence: any PR stacked on another feature branch never receives a Test Suite
run on its head, and manual dispatch cannot help because the audit step gates
every later stage.

| PR | Head | Base | Test Suite | Durability |
|---|---|---|---|---|
| #384 | `73109bea` | `main` | 41233488172 failed at audit (8 executed, 28 skipped) | 41233488181 pass |
| #385 | `81828356` | dep branch | 40918248583 pass (predates current base) | 40918253832 pass |
| #387 | `c2950cd1` | `fix/sms-otp-production-security` | none on head | 41239057229 pass |
| #388 | `113972db` | dep branch | none on head | 41233498123 pass |

#385's green suite is the only run in the stack where Prisma validation, every
overlay installer, Postgres, Redis, the batteries, the route registry and the
backup drill actually executed. Merging #384 first converts it to skipped steps.

## Firebase replacement proof of concept

Branch `exp/firebase-admin-replacement`, head `f7f96c4`, base main `0edefa1d`.
Three files, isolated, not wired into `ssoController` or `firebaseService`.
**68/68 tests pass on that exact SHA.**

- ID token verification: mandatory `auth_time`, `alg`/`kid`/`aud`/`iss`/`sub` checks, expiry at zero skew for firebase-admin parity.
- JWKS: strict per-key validation (non-empty `kid`, RSA, usable modulus and exponent, `use: sig`, `alg: RS256`, 2048-bit minimum), whole-set rejection on any malformed entry, `JWKS_AMBIGUOUS_KEY` on duplicate `kid`, Cache-Control honouring, bounded fetch, single-flight dedupe, one refetch on unknown `kid`.
- FCM: `FCM_UNREGISTERED` requires the exact `type.googleapis.com/google.firebase.fcm.v1.FcmError` type; a bare 404 or a mismatched `@type` is device-state neutral.
- Transport: `AbortController`-based, applied to both the OAuth exchange and the v1 send, proven to terminate a stalled request with a typed error carrying no credentials, one send attempt only.
- OAuth token exchanges deduplicated across concurrent sends.

The PoC does **not** clear the audit and must not be counted as if it did.

## Options for the owner

1. **Allowlist** the three advisories with a documented rationale that the code path is an unused optional dependency. Requires a stated reviewer and expiry.
2. **Gate at `--audit-level=high`** (currently 0 high). Explicitly a policy change, must be recorded as such.
3. **Complete and wire the Firebase replacement**, then remove `firebase-admin`. Highest effort; SSO revocation check and six `firebaseService` call sites are untouched.
4. **Wait upstream** for firebase-admin to move its optional Firestore range onto the 6.x `google-gax` line.

Recommendation, offered not decided: none. Each option changes the meaning of the gate, so this is an owner call. What is factual is that no dependency change reaches zero without an override that has measured worse every time.

## Verified alongside

PIN rotation contract at `c2950cd1`: new PIN exactly four digits, `currentPin`
optional and four digits when supplied, atomic compare-and-swap on `pinHash`
returning `PIN_ROTATION_CONFLICT` 409, and typed refusals
`PIN_REQUIRED`/`PIN_FORMAT_INVALID`/`PIN_CURRENT_REQUIRED`/`PIN_INVALID`/`PIN_LOCKED`/`PIN_NOT_CONFIGURED`.
Conflict handling is not weakened. Client-side handling of the 409s remains
missing in `security_settings.dart`; no Dart SDK is available here, so no
unverified Dart was pushed.

## Not done

- No rebase, retarget, merge, close, deploy or release. No PR bases changed.
- No suppression, no `--force`, no audit-level change, no production wiring.
