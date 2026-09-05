# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-05 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

## 1. Current verified baseline

| Repo | Current main | Current situation |
|---|---|---|
| `AZM-backend` | `b2d74f6bc5b4730ed998e4c42bf1efaf6a7032da` | Tracking concurrency work #220/#221 merged; CI action modernization #223 merged. Financial provider-attempt integrity #222 is open after a failed test-stage run and is not verified. |
| `AZM-businessPortal` | latest verified main includes #81 | Storefront/Studio hardening and API error/multipart contract work are merged; no open PR. |
| `AZM-frontend` | `a1b17dc92671f67ad0b5696ec05029ddbddd650d` | Canonical USDC/GHS FX presentation is enforced across core customer flows; PR #88 updates checkout action version and remains open. |
| `AZM-adminPortal` | `44b3588b5a134b84a528c533e39ba36b9a9c297f` | Withdrawal optimistic mutation concurrency safety is merged; no open PR. |
| `AZM-Planning` | current main | Active loop and current-state navigation have been reconciled to September 5. |

## 2. Verified backend progress since the previous planning snapshot

- PR #205 storefront safe-publish/draft-mutation serialization merged.
- PR #206 Studio geometry/tree ownership validation merged.
- PR #207 safe storefront discovery query correction merged.
- PR #208 removed unsafe storefront route shadows and serialized experience mutation.
- PR #209 repaired render-cache Redis readiness.
- PR #210 invalidated storefront cache on re-enable.
- PR #216 disabled checkout enforcement merged.
- PR #218 public storefront availability gate merged across checkout/products/theme/public-theme/render.
- PR #219 discovery now filters suspended, owner-paused and disabled businesses.
- PR #220 atomically initializes missing `OrderTracking` with unique-order upsert; merged.
- PR #221 serializes tracking mutations per order, creates event timestamps inside the serialization boundary, safely initializes ETA writes, and validates mutation payload types/lengths; merged after exact-head verification.
- PR #223 updates backend Actions checkout/setup-node majors to v7; exact-head run #847 succeeded and the PR is merged.

## 3. Provider-attempt integrity status

PR #222 (`fix(finance): bind provider references to one transaction`) remains **open and unverified**.

The migration `20260830_provider_settlement_attempt_identity` confirms the durable table and unique `(provider, providerReference)` identity are present in the database. The PR correctly attempts to ensure that identity is bound to one canonical `TransactionHistory` row and rejects cross-transaction reuse, including the concurrent `ON CONFLICT` path.

Its latest Actions run completed environment setup, dependency installation, database schema application and Prisma generation successfully, but the `Run tests` step failed. The connector can expose the failed step summary but not the underlying test log text. Therefore the change is intentionally not promoted until the exact failure can be obtained or otherwise deterministically reproduced.

## 4. Cross-repo customer/storefront progress

### Business Portal

Merged storefront/Studio hardening includes responsive breakpoint inheritance, keyboard tile nudging, responsive preview viewports, custom-HTML sanitization, four-column collision-safe tile geometry, session/socket lifecycle cleanup, multi-page tree ownership preservation, business-category policy aliases, deterministic legacy migration IDs, and typed backend error/multipart upload handling. No open Business Portal PR currently exists.

### Flutter customer app

The FX layer now treats canonical `USDC/GHS` retail provenance as authoritative for live conversion and customer display; legacy fields remain compatibility data but are non-canonical. Home market rate/history, rate alerts and Susu display align with the same canonical pair contract. PR #88 is limited to CI action maintenance and is still open.

### Admin Portal

The withdrawal queue uses targeted optimistic updates and rollback protection so stale responses/refetches do not overwrite newer state. Batch approval uses the same concurrency-safe protection. No open Admin Portal PR currently exists.

## 5. Active P0 queue

1. Diagnose and repair backend PR #222; exact-head full CI and required recovery evidence before merge.
2. Finish the canonical POS/invoice tax-authority trace across `BusinessInvoice`, `BusinessInvoiceTaxLine`, `BusinessTaxPreset`, and all producers/consumers before changing legacy tax policy.
3. Complete the cross-client dine-in lifecycle proof: `confirmAndPay`, finalize/payment replay, tips, timeout recovery, reconnect/background ordering, multi-tab races, location/table context, and authoritative customer/business/admin convergence.
4. Continue financial mutation integrity: wallet, escrow, trades, EWA/payroll, orders/invoices, reservations and Admin approvals/releases. For every mutation audit read → decision → write and require atomic/idempotent/state-safe behavior.
5. Complete tenant/actor/state-machine matrix across remaining Business OS resources, reports, notifications, invoices, orders, reservations and administrative actions.
6. Expand Admin financial mutation tests and operational evidence beyond withdrawals.
7. Prove production operations: environment separation, secrets/rotation, migration rollback, observability, worker recovery and reconciliation alerts.
8. Execute concurrency/load/red-team waves only after the critical correctness gates above are green.
9. Finish remaining Experience Blueprint and cross-repo API contract convergence, then move to measured UX/performance and release rehearsal.

## 6. Release definition

AZAMAN is not considered ready because the application builds. Release readiness requires authoritative sources of truth, tenant-safe authorization, atomic/idempotent mutations, enforced state transitions, realtime convergence with authoritative refetch, auditable operations, migration/deployment/rollback evidence, critical Admin safeguards, and demonstrated concurrency/load/adversarial resilience.

## 7. Planning hygiene

Never promote failed or merely opened PRs into the verified baseline. Never recreate work already present on current `main`. After every verified merge, reconcile this file, `ACTIVE_LOOP.md` and `EXECUTION_LEDGER.json` before selecting the next P0. Historical superseded PRs remain historical evidence only.
