# 2026-09-14 CI hardening — verified

## Completed work

### Backend route-integrity enforcement
- Repository: `AzamanLTD/AZM-backend`
- PR: #237 — `chore(ci): enforce backend route registry check`
- Base: `94dfd3144fc3abfd60f51e7a38ad241809b3e8e2`
- PR head: `af79f7b237814a48de7e1944d78e32919df2ccec`
- Merge commit: `6739f9ad9d8bf9d9a9145bde817b5e1b33977725`
- Exact-head workflow: `Azaman Test Suite` run #913 — success
- Change: canonical backend CI now executes `npm run route-check` after the application test suite and before the database recovery drill.
- Scope: CI-only; no application runtime behavior changed.

### Admin Portal dormant contract-suite enforcement
- Repository: `AzamanLTD/AZM-adminPortal`
- PR: #97 — `chore(ci): execute dormant admin lib contracts`
- Base: `730cb18b8173d9309015aa789070633923eede0a`
- Final PR head: `226e41686d3f34eef2661e7cf5c26e4ad7f10945`
- Merge commit: `1dc17ced3f90b43b2e6dc879344af5ba58cbec3e`
- Exact-head workflow: `CI` run #249 — success
- Change: CI now executes the existing `src/lib` Vitest contract suites with a pinned CI-only `vitest@4.1.11` runner.
- Initial CI command defect (`src/lib/*.test.js` treated as a literal filter) was diagnosed from job logs and corrected to `src/lib`; no tests were weakened.
- Scope: CI-only; no application runtime behavior changed.

## Self-audit

Both changes were created directly from current `main`, stayed within the repository change-budget policy, were tested through exact-head CI, and were merged only after verification. The Admin test command was corrected before completion after observing its real CI failure mode.

## Next active work

1. Complete Business Portal Studio Wave C rendered evidence: real overflow/clipping plus responsive phone/tablet/desktop relayout.
2. Begin the next genuinely uncovered Backend financial/control-plane correctness slice from current `main`, not a stale branch.
3. Continue the tenant/state-machine audit in parallel with financial correctness.
4. Keep the deployed four-surface dine-in E2E harness as a bounded integration residual, not a reason to duplicate already-verified component contracts.

## Progress metric

Directional roadmap completion: approximately 30%. This is a weighted engineering-progress indicator, not a production-readiness percentage. Large P0 areas (financial integrity, tenant/state-machine integrity, production operations, red-team and release rehearsal) remain open.
