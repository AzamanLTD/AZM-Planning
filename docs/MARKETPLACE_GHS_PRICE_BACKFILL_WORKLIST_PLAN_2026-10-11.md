# AZM Marketplace — GHS Price Backfill Worklist Plan (P2)

> **Status: PLANNING ONLY (2026-10-11).** No application source code changed; nothing implemented; nothing merged. **P2 implementation is blocked until PR #334 (`feat/ghs-listing-price-foundation`, head `0cf9f372`) passes lead-engineering review and is approved to merge.** This plan documents the sequence so the P2 PR(s) can be drafted the moment #334 lands.
> Companion to `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md` (§4 backfill policy) and `MARKETPLACE_MIX_REDESIGN_2026-10-10.md` (§9). Model ground truth verified against the PR #334 working tree (`prisma/schema.prisma`, `services/businessPriceService.js`) on 2026-10-11.

---

## 1. Purpose and non-negotiable rules

P2 closes the gap between "the BusinessPrice foundation exists" and "the active catalogue carries confirmed GHS prices that Mix and Stores can render." It is a worklist and confirmation workflow, not a migration guess.

1. **No inference, ever.** A GHS amount is NEVER derived from `BusinessProduct.priceRange` buckets, from a legacy `priceUsdc` value, or from any currency conversion (oracle or otherwise). The USDC numeric value is never copied into a GHS field. This is an owner-approved rule of the pricing contract (§4, "Backfill").
2. **Source-backed or nothing.** An imported GHS row exists only when a **genuine source-backed exact-GHS amount already exists outside the platform's USDC data** (§3). A product without such an amount gets **no price row at all** and lands on the merchant input worklist.
3. **Confirmation is the eligibility gate.** Only `confirmPrice` (actor `BUSINESS` or `ADMIN` + timestamp) makes a row eligible. `MIGRATION` rows never confer eligibility. The resolver — `resolveEffectiveGhsPrice` — is the single eligibility gate P3 reads through; nothing else.
4. **Confirmed prices are untouchable by backfill.** Backfill never replaces, archives or modifies a valid confirmed current GHS price (§6).
5. **Legacy USDC unchanged.** `priceUsdc`, `priceRange` and every historical USDC operation are untouched. No USDC value is reinterpreted as GHS.

## 2. Model ground truth (as implemented in PR #334 — no silent schema changes)

- `BusinessPrice.status` supports **`ACTIVE | ARCHIVED` only. There is no `DRAFT` status** in this plan. "Unconfirmed" means: the identity's current `ACTIVE` row (effectiveTo null) has `confirmedAt = null` (typically `confirmedByType: MIGRATION`, `confirmedById = null`).
- `resolveEffectiveGhsPrice` returns the current ACTIVE GHS row **only if** `confirmedByType ∈ {BUSINESS, ADMIN}` AND `confirmedAt != null`. Unconfirmed rows are excluded from every public surface. That exclusion IS the draft semantics.
- `setPrice` **archives the identity's current ACTIVE row before inserting the new one** — it is a replace operation, not an append. Consequence: **backfill must NOT call `setPrice` on an identity that already has a current row** (§6.1); a guarded insert path is required (future implementation work, reviewed with the P2 PR).
- The DB partial unique index (`business_price_current_unique`) enforces exactly one ACTIVE current row per (product, currency, scope, branch); concurrent inserts fail closed.
- This plan introduces **no new status value, no staging model, no schema change**. Any such change would be separate, explicitly reviewed design work.

## 3. Provenance requirements for imported amounts

An exact-GHS amount may be imported only when all of the following hold:

1. **Genuine source.** The amount originates from the business itself or from a business-authorized record — e.g. the merchant's portal-entered bulk import file, a merchant-attested price sheet, or historical business records naming GHS amounts. A USDC price, a price-range bucket or an oracle product is NOT a source.
2. **Exact decimal.** The amount is a canonical exact-2dp GHS string (`/^\d+\.\d{2}$/`, parsed to integer pesewas). Float-shaped input is rejected — never parsed through `Number()`.
3. **Recorded provenance.** The row's `note` carries the backfill batch id, source type (e.g. `PORTAL_BULK_IMPORT`), source reference (file/batch identifier), and an import checksum. A row without provenance metadata fails the backfill's own validation.
4. **Positive amount.** Zero/negative rejected — free-item pricing is unsupported until a separate model is approved (owner-decided).
5. **Unconfirmed by construction.** Imported rows are created with `confirmedByType: MIGRATION`, `confirm = false` → `confirmedAt` null. Backfill can never confirm; confirmation is a human portal action only.

## 4. Inventory and eligibility coverage (the worklist states)

Every active business's active/available product falls into exactly one state, computed through the P1 service surface:

| State | Definition | Customer-visible? | Worklist action |
|---|---|---|---|
| **Confirmed** | current ACTIVE GHS row, `confirmedAt` set, actor BUSINESS/ADMIN | Eligible for Mix/Stores GHS rendering via the resolver | none |
| **Source-backed, unconfirmed** | current ACTIVE MIGRATION row exists (source-backed amount imported), `confirmedAt` null | NOT eligible — excluded by the resolver | merchant confirmation pending: one-tap per product, bulk per section |
| **Missing** | no current ACTIVE GHS row, no source-backed amount | NOT eligible; **excluded from the customer-facing catalogue** unless the business explicitly opted into quotation pricing | merchant price entry required |
| **Quotation opt-in** | missing GHS price AND the business explicitly set the quotation-pricing flag | Stores may display "Price on request" — ONLY because of the explicit opt-in, never inferred from a missing price | still on the worklist: quoting flow, separate design |

Coverage reporting counts products per business per state; the drain rate (unconfirmed → confirmed per day, missing → any) is the admin's progress signal for the P3 readiness gate.

## 5. Merchant-portal price entry and confirmation

- **Worklist surface (business portal):** per business, a price worklist listing (a) source-backed unconfirmed rows with the imported amount, provenance and source, and (b) products with missing GHS prices, with an exact-2dp entry field.
- **Confirmation:** `confirmPrice` only — stamps `confirmedByType`, `confirmedById`, `confirmedAt`, optional note. **Business-level authorization:** the confirming actor must hold an authorized business-portal role for THAT `businessProfileId` (the immutable counterparty identity); an actor of one business can never confirm another business's prices. ADMIN confirmation is a separate audited role, also actor-stamped.
- **Audit trail:** every entry and confirmation records who, when, what amount, from which source. Confirmed rows are never edited — a price change creates a NEW row via `setPrice` (which archives the old), preserving full history.
- **Portal entry is itself source-backed:** a merchant typing a GHS price in their own portal is the genuine source; portal-entered prices may be created directly as confirmed rows (actor = the merchant, `confirmedByType: BUSINESS`).

## 6. Safe, repeatable backfill (conflicts, existing confirmed prices, concurrency, reruns)

### 6.1 The guarded insert (future implementation work — reviewed with the P2 PR)

Because `setPrice` archives the current row, backfill uses a **create-if-absent** path inside one transaction:

1. `SELECT ... FOR UPDATE` (or SKIP LOCKED) the identity's current ACTIVE row for (product, GHS, scope, branch).
2. If a current row exists — **whatever its confirmation state** — **skip** and record the outcome (skip-reason: existing row). Backfill never archives, never replaces, never modifies.
3. If absent: insert the unconfirmed MIGRATION row with provenance note (§3).

The partial unique index is the race fence: a concurrent merchant `setPrice` and a backfill insert cannot both win; the loser gets a unique violation, aborts, and the retry (§6.2) sees the winner's row and skips.

### 6.2 Idempotency and reruns

- Backfill is keyed **per product identity**, not per run: a product already carrying any current row (confirmed or unconfirmed) is skipped forever after.
- Runs are batched (batch id + cursor in `note`); a partial rerun resumes at the cursor and re-processes products whose insert failed transiently. Re-running a completed batch is a no-op by construction.
- Deterministic outcome per product: same input → same single row; never two rows (index-guaranteed), never a replaced confirmed price (guard-guaranteed).

### 6.3 Conflict cases

| Conflict | Resolution |
|---|---|
| Backfill finds confirmed current row | skip — never touched |
| Backfill finds unconfirmed MIGRATION row from an earlier batch | skip — one worklist row per product; reruns never duplicate |
| Merchant sets/enters a price while backfill runs | unique index → one winner; loser skips; merchant's row wins ties (backfill only ever creates-if-absent) |
| Merchant confirms while backfill scans the same product | confirmation only touches unconfirmed rows; backfill only creates absent rows — disjoint mutations, no interleaving hazard |
| Partial batch failure (crash mid-run) | cursor + per-product keys → rerun converges; no orphan duplicates |

## 7. Admin coverage/drain reporting and the P3 gate

- **Coverage report (admin portal):** per business and aggregate — counts of confirmed / source-backed-unconfirmed / missing / quotation-opt-in; drain rate over time; stale worklist entries (unconfirmed > N days).
- **P3 readiness gate (hard):** the discovery API and Mix read GHS pricing **only through `resolveEffectiveGhsPrice`**. No direct table reads, no fallback to `priceUsdc`, no rendering of unconfirmed amounts. Products without a resolved price are excluded from Mix; in Stores they are excluded from the customer-facing catalogue unless quotation opt-in (§4). The coverage report exists precisely to make this gate observable before P3 flips on.
- **Honesty boundary preserved:** nothing in P2 changes `payableIn` or any payment-capability signal. GHS remains list-and-display only until the P6 collection rail ships.

## 8. Real-PostgreSQL test obligations (P2 test matrix)

All run on the CI-identical PG database with the full overlay chain installed. Every test is additive to the 23-test P1 suite (`ghs-price-model.pg.test.js`).

| # | Test | Proves |
|---|---|---|
| T1 | Resolver exclusion | an unconfirmed MIGRATION row (ACTIVE, `confirmedAt` null) is NOT returned by `resolveEffectiveGhsPrice`; the same row after `confirmPrice` IS returned with actor + timestamp |
| T2 | **No USDC reinterpretation** | a product with only `priceUsdc = X` and no source-backed GHS amount receives **no** BusinessPrice row from backfill; the USDC value never appears as/in a GHS amount anywhere (row count 0; no amount equality with X) |
| T3 | **Backfill cannot overwrite a confirmed price** | with a confirmed current row present, backfill run + rerun leaves that exact row untouched (same id, same `confirmedAt`, no ARCHIVED row created) |
| T4 | Idempotent reruns | backfill over the same source twice produces exactly one unconfirmed row per product (no duplicates, same provenance batch lineage recorded) |
| T5 | Concurrent merchant write vs backfill insert | real-PG concurrency on one product: exactly one current ACTIVE row survives; loser (backfill or merchant path) skips cleanly; no partial rows, no archive of a confirmed row |
| T6 | Provenance enforcement | an import without provenance metadata (or with a float-shaped amount, or a non-positive amount) is rejected; no row created |
| T7 | Exact-decimal import | an exact-2dp source string lands as the exact same pesewas value in `amountExact` (round-trip equality), and a float-shaped input is rejected |
| T8 | Coverage report equality | the coverage report's per-state counts equal the actual DB states for a seeded multi-business, multi-state catalogue |
| T9 | Quotation honesty | a missing-price product WITHOUT the explicit quotation opt-in never renders "Price on request" in the Stores surface contract; with opt-in it does — the flag, not the missing price, drives it |
| T10 | Authorization boundary | an actor authorized for business A cannot confirm or enter prices for business B (rejected, audited) |
| T11 | Cross-asset purity | backfill writes only currency="GHS" rows; no USDC-denominated row is ever created by the backfill path |

## 9. Sequencing and dependencies

1. **Now (this document):** plan approved-or-annotated by owner; no code.
2. **Blocked gate:** PR #334 passes lead-engineering review and merges. All P2 work consumes only its schema, overlay and service surface.
3. **P2a — backfill engine + guarded insert:** scan/coverage classification, create-if-absent insert with provenance validation (§3, §6). Idempotent; no public surface. Requires the small `createIfAbsent` service addition — reviewed as its own diff within the P2 PR, since it touches the P1 service.
4. **P2b — merchant-portal worklist + confirmation flow:** entry, one-tap/bulk confirm, authorization, audit trail (§5).
5. **P2c — admin coverage/drain report:** counts, drain rate, stale worklist (§7). May ship with P2b or after.
6. **P3 gate:** discovery GHS rendering switches on only behind `resolveEffectiveGhsPrice` with coverage reported (parent doc phase order preserved).

P2a/b/c may be one PR or sequenced PRs; in all cases the full §8 matrix plus the P1 battery (all suites) must pass on the CI-identical DB before merge review.

— End of plan. Documentation only; no application source code changed, nothing deployed, nothing merged; PR #334 untouched.
