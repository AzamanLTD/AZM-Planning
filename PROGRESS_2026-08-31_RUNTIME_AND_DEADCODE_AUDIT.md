# AZAMAN — 2026-08-31 Runtime Testing + Supersession Audit

## Completed this loop

### Business Portal runtime testing
- PR #18 merged: AuthProvider runtime coverage for business restore/profile loading, fail-closed restore, admin session/business selection, admin login, and logout/realtime cleanup.
- PR #19 merged: TypeGuardedRoute runtime coverage for matching, mismatched, multi-type, and unrestricted business-type routing.
- Both exact heads passed Business Portal CI including smoke test, Vitest, and production build.

### Financial terminal-state regression coverage
- Backend PR #62 is open and based directly on current `main`.
- The suite now covers successful fiat settlement, late-success protection, provider failure reversal, unknown references, wrong transaction types, duplicate failure callbacks, and contradictory late failures after completion.
- Its full PostgreSQL-backed CI run is currently executing.

### Supersession audit
- Admin and Backend `main` now contain only their intended CI workflows after removal of temporary migration workflows.
- Flutter has exactly the intended `flutter-ci.yml` and `android-integration.yml` workflows.
- Flutter direct Socket.IO construction is confined to the canonical singleton `lib/services/socket_service.dart`; no second feature-level socket transport was found.
- Business Portal `FinanceV2.jsx` is intentionally retained: it re-exports `Finance.jsx`, so the apparent old Finance page is still the live implementation.
- Business Portal `patch_pages.py` and `scripts/fix-toast-imports.mjs` were proven to be unreferenced one-off migration utilities and are removed in open PR #20.
- The direct `glob` devDependency remains intentionally parked until it can be removed through a package-manager/lockfile update rather than manual lockfile editing.

## Current next work

1. Finish and merge Backend PR #62 only after its complete PostgreSQL gate is green.
2. Finish Business Portal PR #20 after its CI proves the retired-script removal is safe.
3. Complete the remaining financial event matrix across invoice payment, withdrawal progress/settlement/failure, balance mutation, admin reconciliation exceptions, duplicate/reconnect delivery, and terminal-state ordering.
4. Then continue the marketplace dead-code/supersession audit and whole-system duplication/branch/PR audit.

## Guardrails

- Current `main` is authoritative; stale PR metadata is not.
- Do not resurrect superseded branches.
- Do not create parallel socket/event-bus/state-store architecture.
- Socket events remain convergence signals; canonical APIs/databases remain financial truth.
- Do not remove code merely because it looks old; require entrypoint/reference evidence first.
- Temporary migration workflows must be removed before production-branch completion.
