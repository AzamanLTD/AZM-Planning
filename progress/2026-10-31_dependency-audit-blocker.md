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

Branch `exp/firebase-admin-replacement`, head `c92fb28` (2026-11-04), base main
`0edefa1d`. Six files, all new, isolated, not wired into `ssoController` or
`firebaseService`. `git diff origin/main` on `controllers`, `utils`, `routes`,
`package.json` and `.github` is empty.

**123/123 tests pass locally on that exact SHA** (117 verifier/FCM + 6 SSO
characterisation). No CI test evidence exists and none is obtainable: the audit
step runs before every test step, so any dispatched run skips `Run tests`.

- ID token verification: `alg`/`kid`/`aud`/`iss`/`sub`/`exp` checks, expiry at zero skew for firebase-admin parity, plus mandatory `auth_time` and `iat` (stricter than the SDK, safe for SDK-issued tokens).
- JWKS: per-key validation, whole-set rejection on any malformed entry, duplicate `kid` refusal, Cache-Control honouring, bounded fetch, single-flight dedupe. `use`, `key_ops` and `alg` all follow RFC 7517 optionality.
- FCM: exact FCM error `@type` required for `FCM_UNREGISTERED`; phase-aware timeouts; send-side ambiguity surfaced as `FCM_SEND_OUTCOME_UNKNOWN` (`deliveryUnknown: true`, no automatic retry).
- Credential resolver mirrors production priority (primary var raw-JSON-or-base64, secondary base64 var, module-relative local file), constant parse errors that cannot echo a key, fail-safe boot.
- Flutter click action seeded by default; `uid` alias of `sub` preserved.

### Security defects found by review (all fixed on the branch)

- **Unknown `kid` cost one Google JWKS request per attempt** (unsigned token with a random `kid` was enough). 1:1 egress amplification, and a Google rate-limit would break every real login. Now throttled to one forced refresh per 10s; rotation still works after the interval.
- **No token size bound** before base64 decode and `JSON.parse`. Now 16 KiB.
- **Attacker-controlled `header.alg` echoed into the error message** (size and control characters). Now sanitised and truncated.
- **Local credential path was CWD-relative** while production resolves from the module directory. Now module-relative, `ENOENT` treated as no credential.

## CRITICAL: production SSO dev fallback accepts unsigned tokens

This is **live on main `0edefa1d`** (`controllers/ssoController.js` is byte-identical
between main and #384 head `73109be`), independent of this audit and of the PoC.

`SSO_DEV_FALLBACK=1` decodes a JWT **without verifying its signature**. Proven with
the real handler (only its collaborators stubbed): an `alg: none` token with
`aud: azaman-app` and no `exp` claim produced **HTTP 201, user creation and a session
token pair**.

Reachable only when all hold: `getApps().length === 0`, `NODE_ENV !== 'production'`,
`SSO_DEV_FALLBACK === '1'`, an expected audience configured, and `aud` matching. The
audience guard is not a secret: the Firebase project id ships inside every client
build, so in such an environment the only thing an attacker needs is a public string.

Production is currently safe for exactly one reason: `NODE_ENV=production`
short-circuits to 503.

Why the replacement changes the calculus: the branch is gated on `getApps().length`,
and `require('firebase-admin/app')` is unguarded. Removing the SDK makes that require
throw into the catch and return 401 — accidentally safe. Any edit that makes the
require optional ("no SDK, use the fallback") converts that accident into a live
bypass in every non-production environment.

Proposed remediation, not applied: delete the fallback branch, replace it with a
single readiness check that returns 503 in **all** environments, and verify through
the injected verifier. If local development needs a path, it belongs in a signed
fixture or the emulator, never as a decode in the login handler. The two
characterisation tests are written so the fix is a one-line inversion.

## Options for the owner

No option is recommended here: each changes what the gate means, so this is an owner call.

**A. Keep Firebase Admin and accept the four moderates.** The vulnerable code is never executed (no `getFirestore`/`firestore()` call exists anywhere), so there is no reachable exploit path today and none of the four is high or critical. Audit still counts it because it reads the lockfile. Cost: the gate stays red, so the Test Suite keeps skipping its real stages and #384 stays blocked.

**B. Replace Firebase Admin.** Changes `controllers/ssoController.js` (verify call plus deleting the fallback) and `utils/firebaseService.js` (messaging, credential loading, fail-safe init). Preserves signature, `kid`, `iss`, `aud`, `exp`, `sub` verification and the `uid` contract. Becomes stricter on `auth_time` and `iat`. Improves security **only if the fallback is removed in the same change**. Requires route-level tests, production-caller tests and a green Test Suite. Rollback is a two-file revert plus reinstalling the SDK; the PoC adds no schema or state.

Also unchanged and worth stating: **allowlist**, **`--audit-level=high`** and **wait upstream** remain possible but each is a policy change that must be recorded as such.

The factual finding is that no dependency change reaches zero without an override that has measured worse every time.

## Full-suite reconciliation (base vs PoC head)

Same command, same environment, compared through Jest's structured JSON output
rather than suite names. Base `0edefa1d`: 1,450 passed / 76 failed / 2,136 skipped.
PoC head `cbbe0c1`: 1,534 passed / 76 failed / 2,136 skipped. The 84 extra passes
are exactly the PoC's own tests.

The failing test sets are **identical**, test by test, name for name: 76 tests in
5 suites (`admin-business-delete` 8, `business-ad-adapter` 6,
`business-follower-adapter` 6, `marketplace-v2` 54,
`r15i-kyc-encryption-fail-closed` 2), plus `prod-recovery-guard.pg.test.js`
failing to run on both. Newly failing on the PoC head: none.

Environment, identical on both: no Postgres on `localhost:5432`, no Redis on
`127.0.0.1:6379`, Prisma client generated in both worktrees.

## Verified alongside

PIN rotation contract at `c2950cd1`: new PIN exactly four digits, `currentPin`
optional and four digits when supplied, atomic compare-and-swap on `pinHash`
returning `PIN_ROTATION_CONFLICT` 409, and typed refusals
`PIN_REQUIRED`/`PIN_FORMAT_INVALID`/`PIN_CURRENT_REQUIRED`/`PIN_INVALID`/`PIN_LOCKED`/`PIN_NOT_CONFIGURED`.
Conflict handling is not weakened. Client-side handling of the 409s remains
missing in `security_settings.dart`; no Dart SDK is available here, so no
unverified Dart was pushed.

`verifyIdToken` has exactly one production callsite on `main`:
`controllers/ssoController.js:123`, `getAuth().verifyIdToken(idToken)`, one
argument. No `checkRevoked` flag and no `revokeRefreshTokens` call exists
anywhere in the repository, so revocation checking is not part of the current
behaviour and must not be attributed to it.

## Not done

- No rebase, retarget, merge, close, deploy or release. No PR bases changed.
- No suppression, no `--force`, no audit-level change, no production wiring.

## Update 2026-11-04: the fallback fix is now an open PR

PR #393 (`fix/remove-sso-unsigned-fallback`, head `97189fe8`, base main
`0edefa1d`) removes the unsigned fallback entirely: constant, manual payload
decode, unsigned aud/exp checks and the dev-fallback responses are gone.
`getApps().length === 0` now returns 503 in every environment with no flag able
to change it, and `getAuth().verifyIdToken(idToken)` is the only path to a
`decodedToken`. Two files changed, 14 route-level tests added; 6 of the 14 fail
against main's controller, so the matrix discriminates.

This is deliberately separate from #384 and from `exp/firebase-admin-replacement`
(`844d417`). It needs no dependency decision.

Also corrected: the auth rate limiter does NOT stay disabled until restart. The
`_redisErroring` flag resets on ioredis `ready`. More importantly, production
currently runs Redis OFF (`docs/REDIS_OPERATING_MODES.md`: `REDIS_URL`
deliberately unset after Upstash quota exhaustion), so `_failOpen` never engages
and the auth tier already uses express-rate-limit's in-memory store at 5/min per
IP. The fail-open path is latent, not active.
