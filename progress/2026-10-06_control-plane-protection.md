# Control-plane baseline + branch protection (PR-A of the R42 production-readiness programme)

**Date:** 2026-10-06 UTC
**Authority:** live GitHub API state, verified immediately after configuration.

## Verified repo baseline (before any programme PR merges)

| Repository | main SHA (2026-10-06) | Open PRs | Open issues |
|---|---|---|---|
| AZM-backend | `133dd37bedaa4e89272114ca57ccbf87d58f3b29` | 0 | #271 (oracle stale-rate investigation — active) |
| AZM-frontend | `5331cdb74acd1c5e43501fa5cfaea77e9748209d` | 1 (PR #145, this programme) | 0 |
| AZM-businessPortal | `28fc368c3f0cc624e82f920862149320cf0070d8` | 0 | 0 |
| AZM-adminPortal | `60faed7664879f5f368d114e5941da3b5ad5a5d0` | 0 | 0 |
| AZM-Planning | `73f3871465fb29a84dd8ffa23c74609ca0556bb5` | 10 (overhaul-pack planning mirrors #60–#69) | #52, #53 (+ issue mirrors of PRs) |

Prior to this change, **no repository had branch protection** on `main` (API reported `Branch not protected` on all five).

## Branch protection now enforced (all five repos)

Configured directly via the GitHub API on 2026-10-06 and re-read back as verification:

- `main` cannot be force-pushed.
- `main` cannot be deleted.
- Direct pushes to `main` are blocked (1 approving review required before merge).
- Required CI checks are enforced (per-repo list below).
- Stale review approvals are dismissed when new commits land (`dismiss_stale_reviews: true`).
- Administrators are subject to the same rules (`enforce_admins: true`).

### Required checks per repository

| Repository | Required check-run contexts |
|---|---|
| AZM-backend | `test` (Azaman Test Suite), `financial-durability` (financial-integrity durability lane) |
| AZM-frontend | `quality` (Flutter Quality), `android` (Android Integration) |
| AZM-businessPortal | `validate` (Business Portal CI: Vitest + type/build) |
| AZM-adminPortal | `validate` (CI: Vitest + build/type) |
| AZM-Planning | none (documentation repo; review-only protection) |

## Android PR-gate fix (AZM-frontend PR #145)

The Android Integration workflow historically triggered only on push to `main` + manual `workflow_dispatch`, so PR heads never produced Android evidence and the `android` check could not be required by branch protection.

PR #145 (branch `ci/android-pr-gate`, exact head `5a305bf9243ec6e022a1f732bae9d26d3a3f88b8`) adds `pull_request: branches: [main]` to the **existing** workflow — no duplicate workflow, no job changes, same pinned Flutter `3.47.5` toolchain, same debug-APK artifact contract. Verified live: both `quality` and `android` check runs execute against PR #145's exact head (this is the first PR in the repository's history with an Android check on its own head).

## Notes and deviations

- `required_status_checks.strict` (up-to-date-before-merge) is deliberately `false` for now: enabling it forces constant rebase churn; concurrency safety is instead proven by real-PostgreSQL conditional-claim tests in the backend lanes. Can be revisited when merge-queue usage is formalized.
- Required-review count is 1 (the release owner). CODE_OWNER reviews are not required (no CODEOWNERS file exists yet).
- Merge-queue/merge-group behavior: `flutter-ci.yml` already declares a `merge_group` trigger; merge queue itself is not enabled at the repo level.
- Planning protection requires review but no CI (no CI workflows exist there).
