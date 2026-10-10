# AZM Marketplace — GHS-First Pricing & Payment Contract

> **Companion to `MARKETPLACE_MIX_REDESIGN_2026-10-10.md` (D-1 correction).**
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

**Option B (recommended) — normalized, scoped, effective-dated price rows:**

```
model BusinessPrice {
  id                String   @id @default(uuid())
  businessProductId String
  currency          String   @db.VarChar(3)      // "GHS" today; multi-country-ready
  amountExact       Decimal  @db.Decimal(20,2)  // canonical 2dp; never a float
  scope             String   @default("DEFAULT")// DEFAULT | BRANCH
  locationId        String?                      // null = all branches; set for BRANCH scope
  status            String   @default("ACTIVE") // ACTIVE | ARCHIVED (soft, keeps history)
  confirmedByType   String   @default("BUSINESS")// BUSINESS | ADMIN | MIGRATION
  note              String?                      // audit note (e.g. backfill batch id)
  effectiveFrom     DateTime @default(now())
  effectiveTo       DateTime?                     // null = current
  businessProduct   BusinessProduct @relation(...)
  @@unique([businessProductId, currency, scope, locationId, effectiveFrom])
  @@index([businessProductId, currency, status, effectiveTo])
}
```

- Pros: one row per (product, currency, scope, branch) with exact 2dp; effective dating makes every historical price queryable (order snapshots remain authoritative for what the customer paid, but catalogue history answers disputes); `scope` + `locationId` give branch-specific prices without schema surgery later; `currency` makes the second listing currency a data row, not a migration; `confirmedByType` makes "business-confirmed price" a first-class, auditable fact, which the eligibility rule (§4) depends on.
- Cons: more moving parts than a column; portal product forms write through a service instead of a field.

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
- **No silent fallback to `priceUsdc`.** A product with no business-confirmed GHS price is **excluded from Mix** (it remains visible in its own Store and to the portal worklist) — a missing GHS price must never be presented with a fabricated, converted, or mislabelled figure.
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
3. Merchant accounting: GHS collected is Azaman-platform GHS liquidity (same as deposits today) until a merchant-settlement leg is designed. **Until merchant GHS settlement is separately approved, a GHS-paid order credits the merchant's USDC-equivalent balance via the same quote machinery, with the quote row as the audit bridge — this must be an explicit owner decision (§7), not an assumption.**
4. **Blocker:** none of this exists. Until it ships, checkout remains USDC-escrow (+ POS cash). GHS listing prices must not promise GHS payment.

**USDC checkout (unchanged mechanics, new quote discipline):** escrow funding stays USDC; if a customer pays USDC for a GHS-listed item, the order records the GHS list price + the confirmed USDC quote as a linked, expiring quote row. The GHS list price remains authoritative for the item; the USDC quote is a payment instrument, never a price.

### 3.4 Server revalidation and immutable snapshots

`storefrontCheckoutIntegrityRoutes.js` revalidation extends to GHS: re-read the effective `BusinessPrice` row + `variants`/`modifierGroups` `priceDeltaGhs` **inside the same economic transaction that creates the order**; compute the GHS total server-side from stored rows; reject on any mismatch with what the client displayed (price-changed-between-browse-and-checkout → 409 with the fresh price, never a silent correction). Snapshot fields: `currency`, `amountGhs` (or the legacy `amountUsdc` path unchanged), selected variant/modifier ids **and labels**, per-unit price, quantity, line total, and applicable fees/delivery charges. Snapshots are immutable post-creation (existing behavior for USDC items is preserved; GHS items match it). Refunds return the snapshot's currency and amounts; a GHS-paid order refunds in GHS through the disbursement rail (and vice versa) — refund currency equals payment currency, recorded on the refund row.

### 3.5 Receipts, notifications, accounting, reporting

`BusinessNotification` order copy (today `${amount.toFixed(2)} USDC`, `storefrontRoutes.js`) and customer receipt views read the order's `currency` field and format accordingly (`GH₵ 25.00` / `USDC 25.00`). The merchant notification shows the GHS list total; if funded via USDC quote, both figures with the quote reference. Reporting (admin dashboards reading `amountUsdc` aggregates) is additive: GHS-denominated orders are aggregated separately by currency; no existing USDC report changes meaning.

### 3.6 Contract inventory (do not silently rename)

Existing `priceUsdc`/`amountUsdc` consumers that keep their meaning (non-exhaustive, file-level): backend — `businessProductController`, `hotelMarketplaceController`, `ticketController`, `businessProductService`, `businessService` (`minProductPrice`), `businessOS/hotelOpsService`, `businessOS/posOrderService`, `dineInTabMutationService`, `marketplace/dineInService`, `storefrontProductConfigurationService`, `businessOSRoutes`, `storefrontRoutes`, `storefrontCheckoutIntegrityRoutes`, `install-business-os-overlay`, `prisma/schema.prisma`, `seed-demo`; frontend — `business_models.dart` (`BusinessProduct.priceUsdc`), cart/checkout, marketplace experiences (retail checkout gateway), screens/widgets (18 files); business portal — `Products.jsx`, `HotelRooms.jsx`, `DineInV2.jsx`, `RestaurantInventory.jsx`, preview components. **Rule: no field is renamed, retyped, or reinterpreted. GHS is additive data with its own fields and its own validation path.**

---

## 4. Eligibility, backfill and portal worklist

- A product is **GHS-listing-eligible** iff: effective `ACTIVE` GHS `BusinessPrice` row exists (scope DEFAULT, `effectiveTo=null`) **and** `confirmedByType` is `BUSINESS` or `ADMIN` (never `MIGRATION` — see backfill). Mix discovery requires GHS-listing-eligibility; Stores mode shows the business's own catalogue regardless, but item price display still requires the GHS row (otherwise the item shows "price on request", never a converted figure).
- **Backfill:** a migration job may create `confirmedByType: "MIGRATION"` rows **only as drafts** — e.g. seeded from the business's `priceRange` bucket or portal-entered bulk values — which do **not** confer eligibility. Eligibility turns on when the owner confirms each price in the portal (one-tap confirm flow per product, bulk confirm per section). This makes "business-confirmed GHS price" a real, auditable product requirement rather than a migration guess.
- The portal worklist: "Not yet on Mix" — products without a confirmed GHS price, with inline price entry + confirm, grouped by section, sortable by sales volume. Admin reporting gets the same list at org level.

---

## 5. Migration and rollout plan

1. **Migration (additive only):** create `BusinessPrice`; add `priceDeltaGhs` option to the variants/modifierGroups JSON contract (validated); add `currency` + GHS snapshot fields to order models (nullable, default null = legacy USDC rows unchanged); no column on `BusinessProduct` is touched. Prisma `db push` in dev, SQL migration in CI-identical protocol, all overlays rerun.
2. **Phase order:** (P1) schema + configuration-service validation + portal GHS price entry (no public surface); (P2) backfill drafts + confirm flow + worklist; (P3) discovery API returns GHS pricing + Mix renders GHS (parent doc P1/P2 phases gain this dependency); (P4) order snapshot currency fields + revalidation of GHS totals at checkout (USDC escrow checkout unchanged); (P5) USDC-payment quote discipline (transactionQuoteService purpose for orders); (P6) GHS collection rail per §3.3 — **separate PR, own financial review, blocked on §7 decisions**; multi-merchant checkout stays out of scope per parent doc §5.3.
3. **Compatibility:** existing orders/escrows/reports are byte-identical (all new fields nullable; every read path defaults to legacy USDC behavior when `currency` is null). Financial invariants (FINANCIAL_INVARIANTS.md, durability suite) are untouched until P6, which enters through its own review.

## 6. Test matrix (specified now, written with each phase)

- Exact-decimal: pesewas parser rejects `"25.001"`, `25` (number), `"-0"`, `"25.00 "`; accepts `"25.00"`, `"0"`, `"125000"`. No test constructs GHS via `0.1+0.2`-style float input.
- Price model: effective-row resolution (one ACTIVE row per product/currency/scope); `effectiveTo` archiving; BRANCH rows ignored by v1 resolver; duplicate-row unique violation.
- Eligibility: MIGRATION-drafted rows excluded from Mix; BUSINESS-confirmed included; paused/suspended business still excluded (parent B-1/B-3 rules preserved on the GHS path).
- Variants/modifiers: `priceDeltaGhs` applied additively, 2dp exact; mixed rows (some variants with GHS delta, some without) fail validation for GHS-eligible products.
- Missing GHS price: product with only `priceUsdc` never appears in Mix; appears in worklist; Store detail shows "price on request".
- Browse→checkout drift: price changed between DTO read and order creation → 409 with fresh price (PG test, concurrent update).
- Currency consistency: GHS-paid order snapshot has `currency="GHS"` + exact amounts; USDC path rows have `currency="USDC"`; legacy rows null and read as USDC.
- Retry/idempotency: duplicate pay-intent with same externalRef converges; `UNKNOWN_OUTCOME` leaves order in `AWAITING_PAYMENT` and resolves by webhook; quote expiry refusal (P5/P6).
- Receipt/refund: receipt renders snapshot currency; refund currency == payment currency; merchant notification carries GHS list total.
- Config/regression: storefront checkout integrity suite extended, not replaced; existing USDC snapshots produce identical values.

## 7. Decisions still requiring owner approval

1. **Price model:** normalized `BusinessPrice` (recommended, §1.1) vs additive `priceGhs` column.
2. **Merchant GHS settlement leg:** until a real merchant-settlement design exists, a GHS-paid order either (a) credits merchant USDC-equivalent balance via the order's confirmed quote (recommended — reuses audit machinery, quote row is the bridge) or (b) holds merchant earnings as GHS balance (new GHS liability accounting surface). Blocks P6.
3. **GHS collection rail build:** confirm P6 as a scheduled dependency of any "pay in GHS" product copy. Until it ships, GHS is list-and-display only.
4. **USDC-payment quote source for orders:** approve which rate source(s) the order-quote purpose may read (deposit quotes use the existing authority; an order purpose must be explicitly configured, not inherited).
5. **Backfill policy:** allow MIGRATION-drafted GHS prices (non-eligible until confirmed) vs requiring 100% owner entry. Recommended: drafts allowed, confirmation required.
6. **"Price on request" in Stores:** whether GHS-less products show "price on request" in their own Store (recommended) or are hidden there too.

— End of GHS contract correction. Documentation only; no application source code, PR #332, PR #333 or frontend PR #147 modified.
