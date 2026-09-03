# AZAMAN Current Engineering State

**Last reconciled:** 2026-09-03 UTC  
**Authority:** current repository `main` + exact CI/evidence; this document is the navigation ledger.

> Read `START_HERE.md` → `ROADMAP.md` → this file → `ACTIVE_LOOP.md` → `EXECUTION_LEDGER.json` before engineering.

## 1. Current verified baseline

| Repo | Current verified state | Immediate concern |
|---|---|---|
| `AZM-backend` | Main `301a5795898f4b7de3b69c8156afe027f82e5155`; PR #144 kiosk and #145 POS merged | Close semantic gaps: transaction-time catalog authority, location/table/customer integrity, idempotency payload binding, tax authority |
| `AZM-adminPortal` | Main lineage includes prior PR #95 work | High-risk financial mutation and optimistic-state coverage |
| `AZM-businessPortal` | Main includes merged PR #44 dine-in customer-payment authority fix | Cross-repo `confirmAndPay` reconciliation and Blueprint/runtime contracts |
| `AZM-frontend` | Main includes verified PR #78 payment failure truthfulness | Dine-in retry/realtime convergence |
| `AZM-Planning` | This branch refreshes the canonical continuation files | Merge this reconciliation, then continue implementation P0s |

## 2. Verified backend hardening now on main

- Payroll bounded P2034 retry: PR #131.
- Shift generic lifecycle-status mutation boundary: PR #132.
- Dine-in settlement/finalization/payment-idempotency hardening: PR #135.
- KYB fail-closed gate: PR #136.
- Canonical Business OS Finance routing/runtime repair: PR #137.
- Inventory restock atomicity: PR #141, merged as `a0876b2f61d5bc73acb1a1d76368e019d079fe82`.
- Dine-in customer-order test mock correction: PR #143.
- Kiosk scoped capability + tenant/employee/user/shift/location binding + PIN rate limit: PR #144, merged as `22ee6db6633322bca8be3c60a346f974e5e9323c`.
- POS server-authoritative pricing/payment handling + order/ledger/inventory atomicity + Serializable retry + idempotent replay: PR #145, merged as `301a5795898f4b7de3b69c8156afe027f82e5155`.
- POS duplicate-line recipe-consumption correction is now PR #146 on current main.

## 3. Exact CI evidence

PR #145 head `493d82b32fd8d8d597b6bbccac154a94b2f168e8` passed exact-head Actions run `33777699164`. The run completed the test stage and database backup/restore drill successfully before PR #145 merged.

The backend main branch now points at `301a5795898f4b7de3b69c8156afe027f82e5155`, whose parents are kiosk merge `22ee6db...` and POS merge `493d82b...`. Do not infer CI for the merge commit itself; use the exact PR-head runs as release evidence.

## 4. Active implementation — PR #146

**POS duplicate recipe-line consumption**

- Branch: `fix/pos-recipe-duplicate-line-consumption`.
- Head: `ba4fc6912b35f456cfdde0161a0fb08a36ace230`.
- Root cause: recipe consumption used `new Map(items.map(...))`, so repeated lines for the same product overwrote the earlier quantity and could under-consume ingredients.
- Fix: aggregate quantities by product before multiplying recipe requirements; shared ingredients remain aggregated by inventory item before conditional decrement.
- Regression test covers two lines of the same restaurant product and expects total ingredient consumption for the combined quantity.
- Exact-head CI is required before merge. No CI result is being assumed while the run is pending.

## 5. Immediate P0 queue after PR #146

1. Make POS catalog authority transaction-safe: re-read product state/prices inside the Serializable transaction, and ensure payment totals are derived from that authoritative state.
2. Prove `locationId`/`tableId` ownership and applicability to the business/order; prevent cross-location/table writes through globally supplied identifiers.
3. Prove customer identity semantics for CASH versus AZM/SPLIT and ensure supplied customer IDs cannot cross tenant/business boundaries.
4. Bind idempotency replay to request intent/payload (or explicitly document an API policy that permits key reuse); same key with materially different money/items must not silently replay accidentally.
5. Trace actual tax authority and invoice semantics across Backend and Business Portal before replacing the legacy 2.5% POS tax behavior; add regression coverage against the canonical source.
6. Deep-audit dine-in `confirmAndPay` across Backend → Business Portal → Flutter → Admin visibility, including tab closure, payment authority, tips, idempotency, realtime and timeout recovery.
7. Trace every `updateAccruedWages()` producer/consumer/history before removal or restriction.
8. Strengthen Admin financial mutation and optimistic pending-queue correctness.

## 6. Known residual risks

- POS transaction currently performs price/catalog reads before entering the settlement transaction; this is the highest immediate POS correctness gap after PR #145.
- POS location/table identifiers are persisted in ledger metadata but are not yet proven against business-owned location/table state in this settlement path.
- POS customer identity rules differ by payment method and require explicit tenant/actor evidence.
- Idempotency keys are unique and tenant-checked, but no request-payload fingerprint is stored.
- POS tax is hardcoded to a legacy 2.5% value and has not yet been proven against the canonical tax/invoice authority.
- Kiosk PIN protection is locally rate-limited; distributed/rate-limit topology and timing/enumeration behavior remain a production concern.
- Dine-in realtime/reconciliation and production operations/load/red-team evidence remain open.

## 7. Planning hygiene

Stale Planning PR #27 was closed because it described an obsolete backend authorization batch and was materially behind current main. This branch records the actual post-merge state; after merge, `main` becomes the canonical navigation surface again.
