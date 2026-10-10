# AZM Marketplace — GHS-First Pricing & Payment Contract

> **Companion to `MARKETPLACE_MIX_REDESIGN_2026-10-10.md` (D-1 correction).**
> **OWNER DECISIONS FINALIZED (2026-10-10, on head 7c813fb):** the price model, backfill policy, quotation-pricing opt-in and listing/payment/settlement separation below are owner-approved; the merchant GHS payable direction (§3.3.3) is owner-directed for investigation. §7 records what remains genuinely open. Still documentation-only — no application source code, PR #332, PR #333 or frontend PR #147 modified.
> Documentation and contract correction only, 2026-10-10. No application source code, PR #332, PR #333 or frontend PR #147 was modified.
> Verified against the same heads as the parent document: backend `052370518133c91bdfb119ff271782045194f180`, frontend `9640face067f45d0cfda6bb95f1569af6878add7`. PR #332 (`feat/payment-requests-standalone`, head `b51fa803cd4910fe5bd273c61a6c05b44cfafa01`, 15 files) was inspected for scope only.

**Product requirement (owner-confirmed, supersedes the earlier D-1a default):**
AZM Marketplace serves local businesses in Ghana. The canonical displayed/listing price for Ghana Marketplace products and services is **Ghana cedis (GHS)**. USDC remains a supported payment/funding asset where checkout actually supports it; it is not the default listing currency.

Two rules govern every section below:

- **R1 (decimal truth):** GHS amounts are exact 2-fractional-digit decimals, never binary floating-point money. The repository already has the authoritative pattern: `services/paymentRequestService.js` (PR #332) parses GHS decimal strings into integer pesewas (BigInt) and rejects non-string numeric input as "inherently lossy"; `src/services/fiatLiquidityService.toExactGhsDecimal` is the existing exact-2dp GHS boundary (used by `quoteFiatDepositController.js:253`).
- **R2 (no reinterpretation):** the meaning and numeric values of existing `priceUsdc` / `amountUsdc` fields are unchanged. No USDC value is ever converted into a GHS price, relabelled, or silently mapped. Historical orders, escrow rows, price snapshots and accounting keep their USDC semantics forever.

---

## 1. Price model — recommendation and reasons

### 1.1 Options considered

**Option A — additive column:** `BusinessProduct.priceGhs Decimal(20,2)` (nullable), mirroring `priceUsdc`.

- Pros: smallest migration; one field; trivial queries.
- Cons: cannot express branch-specific prices (`BusinessProduct.locationId` already scopes a product to one branch or all — a price override per branch has no home); cannot express multi-country listings later; no provenance (who confirmed the price, when); variant/modifier deltas would need a parallel JSON field with no validation home; price edits overwrite history silently, so "price at time of order" is only as durable as the order snapshot; it reproduces the exact weaknesses that already exist on `priceUsdc` and pins the second currency in place the same way the first one was.

**Option B (DECIDED — owner-approved 2026-10-10) — normalized, scoped, effective-dated price rows:**

```
model BusinessPrice {
  id                String   @id @default(uuid())
  businessProductId String
  currency          String   @db.VarChar(3)      // "GHS" today; multi-country-ready
  amountExact       Decimal  @db.Decimal(20,2)  // canonical 2dp; never a float; > 0 unless
                                                // a separately approved free-item model permits 0
  scope             String   @default("DEFAULT")// DEFAULT | BRANCH
  locationId        String?                      // null = all branches; set for BRANCH scope
  status            String   @default("ACTIVE") // ACTIVE | ARCHIVED (soft, keeps history)
  confirmedByType   String   @default("BUSINESS")// BUSINESS | ADMIN | MIGRATION
  confirmedById     String?                      // auditable confirming actor (userId / adminId)
  confirmedAt       DateTime?                     // set on every confirmation; part of the audit trail
  note              String?                      // audit note (e.g. backfill batch id)
  effectiveFrom     DateTime @default(now())
  effectiveTo       DateTime?                     // null = current
  businessProduct   BusinessProduct @relation(...)
  @@index([businessProductId, currency, status, effectiveTo])
}
// DB-enforced single-current-price (owner constraint): a PARTIAL unique index, not the
// application layer, guarantees at most one ACTIVE row per price identity at any time:
//   CREATE UNIQUE INDEX business_price_current_unique
//     ON business_prices (business_product_id, currency, scope, COALESCE(location_id, ''))
//     WHERE status = 'ACTIVE' AND effective_to IS NULL;
// Effective-date integrity: effective_to IS NULL OR effective_to > effective_from,
// enforced by a CHECK constraint; archived (effective_to set) rows close exactly one
// active window — no overlapping windows for the same identity.
```

- Pros: one row per (product, currency, scope, branch) with exact 2dp; effective dating makes every historical price queryable (order snapshots remain authoritative for what the customer paid, but catalogue history answers disputes); `scope` + `locationId` give branch-specific prices without schema surgery later; `currency` makes the second listing currency a data row, not a migration; `confirmedByType` makes "business-confirmed price" a first-class, auditable fact, which the eligibility rule (§4) depends on.
- Cons: more moving parts than a column; portal product forms write through a service instead of a field.

**Price-value rules (owner-decided, apply to every write path):**
- A base GHS price must be **positive**, unless a separately approved free-item model explicitly permits zero. No negative or zero base prices without that approval.
- Variant/modifier deltas require **exact-decimal parsing** (the PR #332 pesewas pattern), **bounds** (a configured min/max per delta), and **final-price validation**: base + selected deltas must yield a valid final price (positive, or zero only under the free-item model) — the arithmetic is performed in integer minor units (pesewas), never binary floating-point.
- **No inference:** individual product GHS prices are never derived from `BusinessProduct.priceRange` buckets, from a legacy `priceUsdc` value, or from any conversion. Only source-backed drafts (§4) may enter the system, and only owner confirmation (BUSINESS/ADMIN with actor + timestamp) makes a price eligible.

**Reasons for B:** the owner requirement explicitly asks the model to cover variants, modifiers, branch-specific prices and future multi-country markets. An additive column fails all four structurally, and the failure is a second migration. The one-table cost is modest and the write path is single-sourced. Deciding now also avoids the worst outcome — an additive column that later must be "un-migrated" while live orders reference it.

**Variants and modifier groups** stay where they are — `BusinessProduct.variants` / `modifierGroups` JSON, validated by `services/storefrontProductConfigurationService.js` + `businessProductService.js` (`priceDelta` finite-number check, seed shape `[{label, priceDelta}]`). GHS-eligible products gain an optional **`priceDeltaGhs`** entry per variant/modifier, expressed as an **exact 2dp decimal string** (`"0"`, `"5.00"`, `"-1.50"`), validated by the same configuration service using the PR #332 pesewas parser as the reference pattern. The existing `priceDelta` (USDC) field is untouched; the two deltas are independent data, never converted into each other. Rationale: variants are presentation-time configuration with thousands of existing rows; moving them to relational price rows is a separate, larger refactor that buys nothing for v1 discovery, while the JSON contract gives GHS the same validation surface USDC already has.

**Catalogue resolution rule (v1):** the effective GHS price of an item = the single `ACTIVE` `BusinessPrice` row with `currency="GHS"`, `scope="DEFAULT"`, `effectiveTo=null`. `BRANCH`-scope rows are schema-ready but not consumed by discovery or checkout in v1 (documented, not built).

---

## 2. Payment-rail audit — what actually exists (evidence from code)

Five rails were traced end-to-end. **Conclusion up front: there is no GHS merchant-checkout collection path today.** GHS listing prices can ship in discovery immediately (a confirmed list price is display/comparison data), but "pay this order in GHS" requires the new order-linked collection rail in §3.3. Claiming otherwise in product copy is prohibited.

### 2.1 Rail 1 — Moolre GHS collections (Add Cash): a *deposit* rail, not merchant checkout

`services/moolreCollectionService.js` (309 lines): inbound GHS via PIN-push Mobile Money (MTN/Telecel/AT; `SUPPORTED_CURRENCY = 'GHS'`), MOCK/LIVE gated on `MOOLRE_PROVIDER`. Flow: quote → initiate → status/webhook → reconciliation.

- Quotes come from `src/services/transactionQuoteService.js`: `rateGhsPerUsdc`, `usdcAmount`, `rateSource`, `rateAsOf`, `expiresAt` — a server-authoritative quote with expiry. Exact-2dp GHS boundary is `fiatLiquidity.toExactGhsDecimal` (`controllers/quoteFiatDepositController.js:243-253`).
- Webhook: `POST /api/fiat/webhook/moolre` (`routes/depositRoutes.js:38` → `quoteFiatDepositController.webhook`) verifies the settled GHS against the quote on exact decimals and refuses mismatches; idempotent by externalRef dedup key; typed provider outcomes (`NOT_DISPATCHED` / `DEFINITIVE_REJECTION` / `UNKNOWN_OUTCOME` / `DUPLICATE_REFERENCE`) with the R15-B same-reference retry rule; GHS asset accounting via `fiatLiquidity` receipts.
- **What it settles:** the customer's GHS lands in **Azaman's own Moolre account** (`MOOLRE_ACCOUNT_NUMBER`), and the customer's **USDC wallet** is credited with the quoted `usdcEquivalent`. There is no per-order reference, no merchant account routing, no merchant settlement leg. It cannot be pointed at a merchant checkout by configuration — using it that way is new implementation (§3.3).

### 2.2 Rail 2 — USDC wallet + escrow: the only merchant checkout that exists

`services/escrowService.js` is USDC-only end to end (`amountUsdc` / `feeUsdc` exact Prisma `Decimal`s; release credits `user:${payeeId}:liability` with USDC; fee via `AdminProfitLog` `{amountUsdc}`; ledger rows signed per runDoubleCheck conventions). Storefront checkout: `routes/storefrontCheckoutIntegrityRoutes.js` **already performs server-authoritative price revalidation** — it re-reads `priceUsdc` + `variants` + `modifierGroups` from the DB, computes `configuredUnitPrice(product, item.variants)`, rejects unavailable/inactive products, normalizes the selection, and creates `BusinessOrderItem` rows snapshotting `name`, `unitPrice`, `quantity`, `lineTotal` (Decimal(20,8)). Funding (`fundEscrow` in the frontend `storefront_service.dart`, r42 idempotency keys) uses step-up auth against `/escrow/fund`. Order status mirrors escrow state (`services/businessOrderService.js` `FUNDED→PAID`, `SETTLED/RELEASED→COMPLETED`). POS accepts in-person `paymentMethod: 'CASH'` (`routes/businessOSRoutes.js:4147`), which is not a remote rail.

**Contract gap (critical for this redesign):** `BusinessOrder`, `BusinessOrderItem` and `SmartEscrow` have **no currency field**. Currency is implicit-USDC everywhere (`grep` shows `amountUsdc` in 30 services, 25 controllers, 6 workers). Any GHS order path requires an explicit `currency` on the order snapshot (§3.4).

### 2.3 Rail 3 — Moolre GHS disbursement (withdrawals): an *off-ramp*, not order settlement

`services/moolreDisbursementService.js` (710 lines): GHS payouts to a customer MoMo wallet, async settlement webhook `POST /api/webhook/moolre-disbursement` (`routes/financeRoutes.js:82`). Withdrawals are **USDC-denominated** (`controllers/withdrawalController.js:1781`); the GHS payout amount is derived service-side (`payoutGhs`). A merchant can off-ramp settled USDC earnings to GHS MoMo today, but this is a user-withdrawal rail — nothing links a payout to an order, and the FX spread/rate path is withdrawal accounting, not merchant pricing.

### 2.4 Rail 4 — public rate oracle: exists, and is still not a checkout quote

**Correction to the parent document (its §1.2 said "no public rate endpoint"):** `GET /api/oracle/yellowcard-rate` and `GET /api/oracle/rates` are public, no auth (`routes/oracleRoutes.js`, mounted at `src/routes/index.js:68`; GlobalSettings-backed, 10-minute refresh, source + lastSync exposed). This does **not** change the D-1 conclusion: an oracle/trading rate is a display signal, not a marketplace payment quote. Any USDC-payment quote for an order must go through the `transactionQuoteService` pattern (server-side, rateSource-stamped, expiry, user confirmation), never a raw oracle read.

### 2.5 Rail 5 — PR #332 payment requests: GHS-denominated records, not money movement

`services/paymentRequestService.js` (PR #332, in flight, untouched): standalone requests denominated in exact 2dp GHS decimal strings (`amountExact`, integer-pesewas BigInt, `PAYMENT_REQUEST_MAX_GHS` ceiling). Its own header states "a request is not a money movement". It is a claims/requests feature; it neither collects GHS nor settles to merchants.

---

## 3. End-to-end price consistency architecture

### 3.1 Business portal (price creation/editing/validation)

Product forms (today `src/pages/Products.jsx`, `HotelRooms.jsx`, `DineInV2.jsx`, `RestaurantInventory.jsx`, all writing `priceUsdc` as `Number` with `Math.round(×100)/100` bulk adjust) gain a GHS price field: a **string input validated as exact 2dp decimal** (server-side pesewas parse per the PR #332 pattern; client-side never floats). Saving requires either both currencies or an explicit "GHS only" / "USDC only" choice; the created row is `confirmedByType: "BUSINESS"`. Bulk adjust operates on GHS rows only. Products with no GHS price are flagged in a portal worklist (§5 backfill).

### 3.2 Discovery DTO and Mix comparison

`GET /api/discovery/items` (parent doc §3.1) returns, for Ghana listings:

```json
"pricing": {
  "amount": "25.00",
  "currency": "GHS",
  "displayCurrency": "GHS",
  "hasUsdcListPrice": true
}
```

- `amount` is the **exact 2dp string from the effective `BusinessPrice` GHS row** — string, not number, so no client ever floats it. Comparison/sort in Mix uses these merchant GHS values directly.
- **No USDC→GHS estimate line, ever, as the primary price.** A clearly-labeled estimate is unnecessary for GHS-first listings and remains prohibited as a substitute.
- **No silent fallback to `priceUsdc`.** A product with no business-confirmed GHS price is **excluded from Mix and from the customer-facing Stores catalogue** (owner-decided; "Price on request" renders only on explicit quotation-pricing opt-in — §4). A missing GHS price must never be presented with a fabricated, converted, or mislabelled figure.
- `hasUsdcListPrice` is informational only (e.g. to hint at funding options at detail level, never to price the item).
- `priceDeltaGhs` on variants/modifiers surfaces in the DTO as 2dp strings.

### 3.3 Order payment — the honest capability split

The order-time contract separates five currencies that today are all silently "USDC":

1. **List currency** — GHS (`BusinessPrice` row).
2. **Customer-selected payment currency** — GHS (via the new rail below) or USDC (existing escrow), or in-person cash at POS.
3. **USDC quote (when paying by USDC)** — server-side quote from `transactionQuoteService` machinery: approved rate source, `rateSource` + `rateAsOf`, hard expiry, user confirmation step. **Never a trading quote, oracle read, client-side rate, or invented number.**
4. **Provider settlement currency** — GHS in Moolre's rails; USDC in the wallet/escrow rail.
5. **Recorded order currency** — explicit `currency` field on `BusinessOrder`/`BusinessOrderItem` snapshot: `"GHS"` if a GHS collection paid it, `"USDC"` if escrow funded it. Receipts, notifications, refunds and reporting read this field; no component infers currency from amounts.

**GHS checkout (NEW, explicit dependency):** order-linked Moolre collection is *not buildable from configuration*. The required implementation, reusing Rail 1's verified machinery (quote service, `toExactGhsDecimal`, typed provider outcomes, externalRef dedup, webhook verification, `fiatLiquidity` receipts):

1. `POST .../orders/:id/pay/ghs` — creates an order-scoped collection intent (externalRef derived from orderId + idempotency key; amount = revalidated server-side GHS total, §3.4), returns payment instructions.
2. Webhook extension resolves the reference to the **order**, verifies settled GHS == quoted GHS on exact decimals, and transitions `AWAITING_PAYMENT → PAID` durably (idempotent on externalRef; `UNKNOWN_OUTCOME` never fails the order — status/callback resolves under the same reference, R15-B).
3. Merchant accounting — **owner-directed design direction (2026-10-10):** the preferred local-market design is **GHS merchant payable accounting with supported merchant payouts**. This is an investigation workstream (§3.3.3), not an implementation assumption. **Merchant proceeds must NOT be automatically converted to USDC without explicit owner approval and a separately reviewed financial contract.** GHS collected by an order is platform GHS liquidity until the merchant-settlement leg is designed and approved; no interim auto-conversion may be inserted as a shortcut.
4. **Blocker (hard dependency, owner-confirmed):** none of this exists. GHS checkout — collection, callback, reconciliation, ledger and merchant settlement — is a hard dependency before any product copy claims customers can pay orders in GHS, and it stays behind its own financial review. Until it ships, checkout remains USDC-escrow (+ POS cash). GHS listing prices must not promise GHS payment. No invented payment success.

**3.3.3 Merchant GHS payable investigation (pre-P6 workstream, design document — no code):** specify (a) the merchant payable ledger account per currency (GHS-native payable, not a USDC proxy); (b) supported merchant payout rails (Moolre disbursement to merchant MoMo, bank transfer if/when a provider exists) with their fees, settlement timing and reconciliation; (c) the fee/commission model in GHS (exact 2dp, same decimal discipline); (d) refund/dispute handling per currency; (e) the financial-invariants amendment that the durability suite must pin before any GHS settlement code merges. Output: a financial contract reviewed under the same rigor as FINANCIAL_INVARIANTS.md, then owner approval, then P6 implementation.

**USDC checkout (unchanged mechanics, new quote discipline):** escrow funding stays USDC; if a customer pays USDC for a GHS-listed item, the order records the GHS list price + the confirmed USDC quote as a linked, expiring quote row. The GHS list price remains authoritative for the item; the USDC quote is a payment instrument, never a price.

### 3.4 Server revalidation and immutable snapshots

`storefrontCheckoutIntegrityRoutes.js` revalidation extends to GHS: re-read the effective `BusinessPrice` row + `variants`/`modifierGroups` `priceDeltaGhs` **inside the same economic transaction that creates the order**; compute the GHS total server-side from stored rows; reject on any mismatch with what the client displayed (price-changed-between-browse-and-checkout → 409 with the fresh price, never a silent correction).

**Snapshot fields — three separate facts, three separate field groups (owner rule; one `currency` field must never mean all three):**
1. **Listing-price snapshot (agreed at order time):** `listingCurrency` ("GHS"), `listingAmountGhs` exact 2dp, per-item `unitPriceGhs`, selected variant/modifier ids **and labels**, quantity, line totals, applicable fees/delivery charges. Immutable post-creation.
2. **Payment record (what actually moved):** `paymentAmount`, `paymentCurrency`, provider/reference; for USDC payment additionally the server-confirmed quote identity, quote rate source, quote timestamp and quote expiry. A GHS-listed order paid in USDC carries both facts independently — the GHS listing snapshot is never overwritten by the payment record.
3. **Merchant payable (what the merchant is owed):** `payableAmount`, `payableCurrency` (GHS-native per §3.3.3 once approved), settlement references/ids. Separately represented and separately mutated; refund flows read the *payment* group (refund currency == payment currency), settlement flows read the *payable* group, and reporting reads the *listing* group.

Legacy USDC-only rows keep their existing fields and behavior unchanged (nullable new columns, null = legacy USDC path). Refunds return the snapshot's currency and amounts; a GHS-paid order refunds in GHS through the disbursement rail (and vice versa) — refund currency equals payment currency, recorded on the refund row.

### 3.5 Receipts, notifications, accounting, reporting

`BusinessNotification` order copy (today `${amount.toFixed(2)} USDC`, `storefrontRoutes.js`) and customer receipt views read the order's `currency` field and format accordingly (`GH₵ 25.00` / `USDC 25.00`). The merchant notification shows the GHS list total; if funded via USDC quote, both figures with the quote reference. Reporting (admin dashboards reading `amountUsdc` aggregates) is additive: GHS-denominated orders are aggregated separately by currency; no existing USDC report changes meaning.

### 3.6 Contract inventory (do not silently rename)

Existing `priceUsdc`/`amountUsdc` consumers that keep their meaning (non-exhaustive, file-level): backend — `businessProductController`, `hotelMarketplaceController`, `ticketController`, `businessProductService`, `businessService` (`minProductPrice`), `businessOS/hotelOpsService`, `businessOS/posOrderService`, `dineInTabMutationService`, `marketplace/dineInService`, `storefrontProductConfigurationService`, `businessOSRoutes`, `storefrontRoutes`, `storefrontCheckoutIntegrityRoutes`, `install-business-os-overlay`, `prisma/schema.prisma`, `seed-demo`; frontend — `business_models.dart` (`BusinessProduct.priceUsdc`), cart/checkout, marketplace experiences (retail checkout gateway), screens/widgets (18 files); business portal — `Products.jsx`, `HotelRooms.jsx`, `DineInV2.jsx`, `RestaurantInventory.jsx`, preview components. **Rule: no field is renamed, retyped, or reinterpreted. GHS is additive data with its own fields and its own validation path.**

---

## 4. Eligibility, backfill and portal worklist

- A product is **GHS-listing-eligible** iff: effective `ACTIVE` GHS `BusinessPrice` row exists (scope DEFAULT, `effectiveTo=null`) **and** `confirmedByType` is `BUSINESS` or `ADMIN` with an auditable confirming actor + timestamp (never `MIGRATION` — see backfill) **and** the base price is positive (zero only under a separately approved free-item model). Mix discovery requires GHS-listing-eligibility.
- **Stores visibility (owner-decided):** an unpriced item stays **out of the customer-facing catalogue entirely** (Mix and Stores alike) until a confirmed GHS price exists. "Price on request" is displayed **only when the business explicitly opts the item into quotation-based pricing** (NEW per-item opt-in flag) — a missing price never implies quotation intent. Opted-in quotation items render "Price on request" with an ENQUIRE action; everything else unpriced is hidden from customers and surfaces in the portal worklist.
- **Backfill (owner-decided):** source-backed migration records remain **drafts** until owner confirmation. **No inference:** drafts are never derived from `priceRange` buckets and never converted from `priceUsdc` — only source-backed values (e.g. a business's own portal-entered bulk import) may enter as `MIGRATION` drafts, which do **not** confer eligibility. Eligibility turns on when the owner confirms each price in the portal (one-tap confirm flow per product, bulk confirm per section; each confirmation records actor + timestamp). This makes "business-confirmed GHS price" a real, auditable product requirement rather than a migration guess.
- The portal worklist: "Not yet on Mix" — products without a confirmed GHS price, with inline price entry + confirm, grouped by section, sortable by sales volume. Admin reporting gets the same list at org level.

---

## 5. Migration and rollout plan

1. **Migration (additive only):** create `BusinessPrice` **with the DB-enforced constraints of §1.1** (partial unique index on one ACTIVE current row per product/currency/scope; effective-date CHECK; confirming actor + timestamp columns); add `priceDeltaGhs` option to the variants/modifierGroups JSON contract (validated, with bounds and final-price validation in integer pesewas); add the three separated field groups to order models — listing snapshot, payment record, merchant payable (nullable; default null = legacy USDC rows unchanged) — never one overloaded `currency` field; add the per-item quotation-pricing opt-in flag; no column on `BusinessProduct` is touched. Prisma `db push` in dev, SQL migration in CI-identical protocol, all overlays rerun.
2. **Phase order:** (P1) schema + configuration-service validation + portal GHS price entry (no public surface); (P2) backfill drafts + confirm flow + worklist; (P3) discovery API returns GHS pricing + Mix renders GHS (parent doc P1/P2 phases gain this dependency); (P4) order snapshot currency fields + revalidation of GHS totals at checkout (USDC escrow checkout unchanged); (P5) USDC-payment quote discipline (transactionQuoteService purpose for orders); (P6) GHS collection rail per §3.3 — **separate PR, own financial review; GHS checkout is a hard dependency for any pay-in-GHS claim**, blocked on the merchant-payable investigation (§3.3.3) and the remaining §7 items; multi-merchant checkout stays out of scope per parent doc §5.3. **Pre-P6:** the merchant GHS payable investigation is a design-document workstream — its financial contract must be reviewed and owner-approved before any P6 code.
3. **Compatibility:** existing orders/escrows/reports are byte-identical (all new fields nullable; every read path defaults to legacy USDC behavior when `currency` is null). Financial invariants (FINANCIAL_INVARIANTS.md, durability suite) are untouched until P6, which enters through its own review.

## 6. Test matrix (specified now, written with each phase)

- Exact-decimal: pesewas parser rejects `"25.001"`, `25` (number), `"-0"`, `"25.00 "`; accepts `"25.00"`, `"0"`, `"125000"`. No test constructs GHS via `0.1+0.2`-style float input.
- Price model constraints: effective-row resolution (one ACTIVE row per product/currency/scope); `effectiveTo` archiving; BRANCH rows ignored by v1 resolver; **second concurrent ACTIVE row per product/currency/scope rejected by the DB partial unique index (not just app code); `effectiveTo ≤ effectiveFrom` rejected by the CHECK constraint; overlapping active windows impossible**; confirmation records actor + timestamp (assert on confirm flow).
- Price-value rules: **non-positive base price rejected** (zero accepted only under an approved free-item model flag); `priceDeltaGhs` exact-decimal parsing, bounds enforcement, and final-price validation (base + deltas in integer pesewas; float input paths rejected).
- Eligibility: MIGRATION-drafted rows excluded from Mix; BUSINESS-confirmed included; paused/suspended business still excluded (parent B-1/B-3 rules preserved on the GHS path).
- Variants/modifiers: `priceDeltaGhs` applied additively, 2dp exact; mixed rows (some variants with GHS delta, some without) fail validation for GHS-eligible products.
- Missing GHS price: product with only `priceUsdc` never appears in Mix; appears in worklist; **hidden from the customer-facing Stores catalogue too, unless explicitly opted into quotation pricing — in which case it renders "Price on request" with ENQUIRE; a mere missing price never renders "Price on request" (opt-in only, owner rule)**.
- Browse→checkout drift: price changed between DTO read and order creation → 409 with fresh price (PG test, concurrent update).
- Currency separation (owner rule): listing snapshot, payment record and merchant payable are distinct field groups on the order path — a GHS-listed order paid in USDC carries `listingCurrency="GHS"` + `paymentCurrency="USDC"` + the quote identity/source/timestamp/expiry, each group mutated only by its own flow; merchant payable fields are independent of both; legacy rows null and read as USDC everywhere.
- Payment capability honesty: `payableIn` in the public DTO reflects only actually-supported rails; no "GHS" value until the collection rail ships (no invented payment success).
- Retry/idempotency: duplicate pay-intent with same externalRef converges; `UNKNOWN_OUTCOME` leaves order in `AWAITING_PAYMENT` and resolves by webhook; quote expiry refusal (P5/P6).
- Receipt/refund: receipt renders snapshot currency; refund currency == payment currency; merchant notification carries GHS list total.
- Config/regression: storefront checkout integrity suite extended, not replaced; existing USDC snapshots produce identical values.

## 7. Decision status (updated 2026-10-10, owner decisions on head 7c813fb)

**Decided by the owner:**
1. **Price model: APPROVED** — normalized `BusinessPrice` (§1.1), with the owner-added constraints: DB-enforced uniqueness for the single current active effective price per product/currency/scope, effective-date integrity, auditable confirming actor + timestamp. `priceUsdc` and all historical USDC operations unchanged.
2. **Price-value rules: DECIDED** — positive base price (zero only under a separately approved free-item model); exact-decimal delta parsing with bounds and final-price validation; no binary floating-point; no inference from `priceRange` or from `priceUsdc`.
3. **Backfill policy: DECIDED** — source-backed migration records remain drafts until owner confirmation; drafts never confer eligibility.
4. **"Price on request" in Stores: DECIDED** — explicit quotation-pricing opt-in only; otherwise unpriced items are hidden from the customer-facing catalogue until confirmed and exposed in the portal worklist.
5. **Listing/payment/settlement separation: DECIDED** — three separately represented facts on every GHS-listed order (§3.4); no overloaded `currency` field.
6. **Merchant proceeds direction: DECIDED (direction, not contract)** — GHS merchant payable accounting and supported merchant payouts are the preferred local-market design and are directed for investigation (§3.3.3); merchant proceeds must NOT be automatically converted to USDC without explicit approval and a separately reviewed financial contract.
7. **GHS checkout: DECIDED** — hard dependency before any pay-in-GHS claim; collection, callback, reconciliation, ledger and merchant-settlement work stay behind their own financial review.

**Still genuinely open (blockers):**
1. **Merchant GHS payable financial contract** — the §3.3.3 investigation must produce a reviewed, owner-approved contract before P6 settlement code. Blocks P6.
2. **GHS collection rail scheduling** — the rail is a confirmed hard dependency but has no schedule yet; until it ships, GHS is list-and-display only and `payableIn` never reports "GHS".
3. **USDC-payment quote source for orders** — which rate source(s) the order-quote purpose may read still needs explicit approval (deposit quotes use the existing authority; an order purpose must be explicitly configured, not inherited).
4. **Free-item (zero-price) model** — needs separate, explicit approval before any zero-price listing is permitted.

— End of GHS contract correction. Documentation only; no application source code, PR #332, PR #333 or frontend PR #147 modified.
