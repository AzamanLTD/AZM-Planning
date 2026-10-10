# AZM Marketplace Mix Redesign — Architecture & Product Contract

> **Agent A, Phase 0 (documentation-only).** Verified against live main on 2026-10-10:
> frontend `9640face067f45d0cfda6bb95f1569af6878add7`, backend `052370518133c91bdfb119ff271782045194f180`.
> Open PRs #147 (frontend, receive-request-recent-interaction, head `3a656d8`) and #332 (backend, payment-requests-standalone, head `b51fa80`) were inspected for scope only; neither is modified by this phase.
> **D-1 CORRECTION (2026-10-10, owner-confirmed):** GHS is the canonical listing currency for Ghana Marketplace — see §3.4 and the companion contract `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md` (price model, payment-rail audit, migration and tests). The earlier USDC-default D-1a below is superseded; all D-1-dependent sections were revised in the same pass.
> **OWNER DECISIONS FINALIZED (2026-10-10, on head 7c813fb):** D-2 through D-8 and the GHS-pricing follow-ups are decided — §9 records each outcome; every dependent section (§3.1, §3.4, §4, §5, §6, §7, §8) was reconciled in the same pass, including the audited `storefrontDisabled` semantics (§9 D-5). Genuine remaining blockers are listed in §9. Still documentation-only; no application source code changed.
> This document supersedes the merchant-card-era plan in `overhaul-pack/02_MARKETPLACE_DISCOVERY.md` (branch `pack/02-marketplace-discovery`, not modified). No application source code changed in this phase.

Mission: redesign AZM Marketplace around **customer-led, cross-business discovery**. The item — not the business card — becomes the primary visual subject of a new **Mix** browse mode, while **Stores** remains the deliberate single-business mode. This is a product-direction document: every claim about what exists today is evidence-backed with a file path; everything new is labeled **NEW** and needs an approval decision listed in §9.

---

## 1. Current-state audit (evidence)

### 1.1 Frontend — marketplace discovery today (main 9640face)

**Portal structure** — `lib/screens/marketplace/marketplace_home_screen.dart` (1322 lines): portal mode renders `DiscoveryHeader`, `IntentRail`, `WorldDeck`, featured rails and an explore-all entry; explore mode is a business list driven by one search state. The current design is **business-card-first**: the smallest discovery unit the user browses is a business (`MerchantCard`), never an item.

**Intent rail** — `lib/widgets/marketplace/discovery/intent_rail.dart:41-47`: categories Eat/Shop/Ride/Stay map to `AzIntent` + `worldWire` values `FOOD_BEVERAGE`, `RETAIL`, `LOGISTICS`, `HOSPITALITY`. Nearby/My saves/Recent are mode flags on the same rail. **Mix/Stores as a discovery-mode control does not exist — NEW.**

**Merchant card** — `lib/widgets/marketplace/discovery/merchant_card.dart`: 220px-wide business card; cover + logo hero, `TrustMark`, relevance line (open/closed via `business.openStateAt(s.now)`, distance km), one primary action label, save toggle. No numerical stars on the card — good precedent for the honest-signal rules below.

**Rating sort still exists** — `lib/widgets/marketplace/discovery/world_deck.dart:70` sorts worlds by `averageRating` descending. Mission removes numerical-star and Top-Rated-driven default ordering; this line and the backend `trending` sort (§1.2) are the two places to change.

**Discovery snapshot** — `lib/providers/marketplace_discovery_provider.dart` + `lib/experience/gateways/marketplace_discovery_gateway.dart`: one derived `DiscoverySnapshot` (catalog, featured, distance map, saved ids, injectable clock). `supportedSignals` honestly reports only `{openNow, nearby, topRated}` and deliberately does not fabricate `recentlyAdded/departingSoon/availableTonight`. The gateway pattern is reusable for Mix: **NEW** `MarketplaceMixGateway` should follow the same "capability truth" rule.

**Search** — `lib/providers/marketplace_search_provider.dart`: ONE search state; commits persist recent searches and push into `businessSearchProvider.search(q, category: committedWorldWire)`. Search hits businesses by name/bizId/description (backend §1.2) — **there is no item-name search today.**

**Item data (client model)** — `lib/models/business_models.dart:563` `BusinessProduct`: `priceUsdc` (double), `isActive`, `isAvailable` not parsed (only backend truth), `category` free string, `locationId`, `tags`, `totalOrders`, `totalRevenue`, `deliveryTerms`, `estimatedDelivery`. Backend `variants`/`modifierGroups` JSON (§1.2) are **not modeled client-side** — variant-aware display is NEW work.

**Cart & checkout** — `lib/providers/cart_provider.dart:96` `CartState` carries exactly one `businessProfileId`/`businessName`; switching business clears it. `cart_screen.dart` `_placeOrder` targets one business. **Single-business cart is a hard boundary; multi-merchant cart is out of scope for this phase** (§5.3 describes the future architecture only).

**Routing & web** — `pubspec.yaml` go_router ^14.8.1; canonical routes in `lib/router/app_router.dart`; deep links are **`azaman://` custom scheme only** (`lib/router/deep_link.dart`, `kAzamanDeepLinkScheme`). There are **no universal/app links, no web fallback, no web build, no business slugs** — public web pages (§6) are entirely NEW.

**Item-level signals that already exist:** `lib/providers/recently_viewed_products_provider.dart` (recently viewed items) — a real seed for "For You" personalization and resume cards.

### 1.2 Backend — discovery, catalog, analytics today (main 05237051)

**Models** (`prisma/schema.prisma`):
- `BusinessProfile` (line 2709): `bizId` unique, `category` enum, `subcategory` (wire value), `priceRange` 1-4, `cuisineTypes`, `amenities`, `businessMeta` Json, `isVerified`, `isSuspended`, `isPausedByOwner`, `groupId`, denormalized `averageRating`/`reviewCount`/`totalEscrows`/`totalVolume`. `website` field holds a business's own external site — not an Azaman-hosted page.
- `BusinessProduct` (line 3171): `priceUsdc Decimal(20,8)` is the **only** price; `isActive` (soft delete) + `isAvailable` (sold-out toggle); `catalogSectionId`; `locationId` (null = all branches); `preparationMins`; `tags String[]` (incl. `POPULAR`/`NEW` — currently owner-declared, not evidence-backed); `sku`/`barcode`/`stockQty`/`lowStockThreshold` (retail); `variants`/`modifierGroups` Json; `totalOrders`/`totalRevenue` denormalized. Indexes cover per-business lookups only (`[businessProfileId]`, `[businessProfileId, isActive]`, `[slug]`, `[catalogSectionId]`). **No cross-business index on category/price/tag — Mix queries would table-scan; NEW indexes required.**
- `CatalogSection` (line 3144): free-form `name` ("Starters", "Main Course", "Drinks"), optional `locationId` (null = all branches), `availableFrom/To` time windows, `displayOrder`, `isActive`.
- `BusinessLocation` (line 2899): lat/lng, `operatingHours` map, `isPrimary`, `isActive`.
- `BusinessReview` (line 3054): integer rating 1-5, one per order/invoice/reservation/transitBooking (`orderId @unique` etc.), `sourceType`. Real review data exists — usable as a business-level trust signal only.
- `StorefrontAnalyticsEvent` (line 6000): `businessProfileId`, `eventType` string, `metadata` Json, indexes `([businessProfileId, eventType])`, `([createdAt])`. **Business-level only: no item id, no session/visitor identity, no impression semantics, no dedup, single-row inserts.**

**Discovery endpoints (all business-level; no cross-business item API exists):**
- `GET /api/business/search` — `controllers/businessController.js:84` → `services/businessService.js:124` `searchBusinesses`: cursor pagination, take ≤ 50; filters category/subcategory/priceRange/verified/minRating; **filters `isSuspended: false` but NOT `isPausedByOwner`** (audit finding B-1); sort default `totalEscrows desc`, `trending` → `utils/ranking.js` `discoveryScore` = Wilson lower bound on averageRating + `log10(totalEscrows+1)*0.05` (audit finding B-2: star-driven); B-12 attaches `minProductPrice` via groupBy. Hardcoded `EXCLUDED_NAMES` test-account filter.
- `GET /api/business/search/nearby` — public, `businessLocationController.searchNearby`, returns per-location `distanceKm`.
- `GET /api/storefront/discover` — `routes/storefrontRoutes.js:86`: published `BusinessStorefrontLayout` tiles. **Audit finding B-3: the category filter is nested wrong (`businessProfile: { businessProfile: { category } }`) and can never match; also this endpoint filters `isPausedByOwner` while business search does not — the eligibility rule must be unified.**
- `GET /api/business/:bizId/products`, `/:bizId/menu` (sections + products), `/:bizId/locations`, `/:bizId/reviews` — public per-business reads.
- `GET /api/business/products/:productId` — public single-product read.

**Orders/bookings:** `POST /api/storefront/:businessProfileId/order` (`routes/storefrontRoutes.js:749`): auth required, single `productId` + quantity, `idempotencyKey` with request-hash replay/conflict guard, enforces `isSuspended` (404) and `isPausedByOwner` (400) and product `isActive && isAvailable`, creates `BusinessOrder` status `AWAITING_PAYMENT`, linked to escrow. Hotel reservations: `POST /marketplace/business/:bizId/reservations`. Transit: `POST /marketplace/transit/trips/:id/book` (idempotency middleware). All order/booking models are **single-business**; escrow (`services/escrowService.js`) settles in USDC.

**Analytics ingestion:** `POST /api/storefront/:businessProfileId/events` (`routes/storefrontRoutes.js:422`): public optional-auth, allowlist (`storefront_view, widget_view, cta_click, product_tap, follow_click, review_click, message_click, share_click`), free-form `metadata` Json, enriches `viewerId`. **One insert per request — no batching; `product_tap` exists but no impression, viewability, or outcome events.**

**Currency:** today every product/order/escrow amount is USDC (`priceUsdc`, `amountUsdc`); there is no product-level GHS price yet. Correction of the original audit: a public rate endpoint DOES exist (`GET /api/oracle/yellowcard-rate`, `GET /api/oracle/rates`, public per `routes/oracleRoutes.js`, mounted `src/routes/index.js:68`) — it is an oracle/trading display rate, NOT a marketplace quote source, and remains unusable for pricing. Full payment-rail audit (Moolre collections = deposit rail into Azaman's own account crediting the USDC wallet; escrow = USDC-only merchant checkout with no currency field on order snapshots; Moolre disbursement = USDC-denominated user off-ramp; no GHS merchant-checkout collection path exists): see the companion `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md` §2. **Canonical listing currency is GHS — see §3.4 (D-1, corrected).**

**Public web/SEO:** none. No sitemap, robots, SSR, or hosted public page anywhere in the backend; `GET /api/storefront/:id/render` and `/public-theme` return JSON intended for a "web ordering integration" surface that does not exist yet. Business portal (`AZM-businessPortal`, React/Vite SPA) and admin portal are auth-gated management UIs. **§6 is greenfield.**

**Open PRs (scope only, untouched):** #332 backend payment-requests (schema + controller + routes; additive, no marketplace files); #147 frontend payment-requests UI (`app_router.dart`, `route_registry.dart`, home widgets). No file overlap with the Mix surface; sequence note in §7.

### 1.3 What exists vs. what is missing

| Capability | Status |
|---|---|
| Business-level search/browse with eligibility filters | EXISTS (with findings B-1..B-3) |
| Per-business product/menu reads | EXISTS |
| **Cross-business item-level discovery API** | MISSING — NEW (§3) |
| Mix/Stores mode UI | MISSING — NEW (§2) |
| Canonical taxonomy mapping from free-form sections | MISSING — NEW (§4) |
| Item-level analytics (impressions, outcomes) | MISSING — NEW (§5) |
| Star-free ranking with diversity/fair exposure | MISSING — NEW (§5.2) |
| Public web business pages / slugs / SEO | MISSING — NEW (§6) |
| Multi-merchant checkout | MISSING — future architecture only (§5.3) |

---

## 2. Product flows and recommended Marketplace layout

### 2.1 Modes, not categories

Mix and Stores are a **discovery-mode control** sitting above the category rail, defaulting to Mix. Both retain the selected category and search text when switching (the single `marketplaceSearchProvider` state already carries the category commit; NEW: add `discoveryMode { mix, stores }` to it, and to `MarketplaceSearchBinding`).

- **Mix:** one continuous, paginated feed of *items* from all eligible businesses matching the category/search/filter context. Item cards are the primary subject.
- **Stores:** today's business-level experience (portal rails, world deck, merchant cards, business list) — preserved as-is behind the toggle.

### 2.2 Eat → Mix: the first full example

**Canonical browse filters** (chips inside Eat/Mix): For You · Meals · Starters · Sides · Drinks · Desserts.

**Taxonomy mapping, not section-name enforcement** (details §4): each `CatalogSection` name is mapped to a canonical filter via a deterministic mapping table (`catalog_section_taxonomy`), with an owner-confirmable suggestion in the business portal and a safe `Unmapped` bucket that only surfaces under "Meals" for Eat (never dropped silently). Section time windows (`availableFrom/To`) and product `isAvailable` are authoritative: a section closed at browse time excludes its items from that filter's feed.

**Item card (NEW `MixItemCard`) — required fields, honest-only:**

| Field | Source | Honesty rule |
|---|---|---|
| Item name + primary image | `BusinessProduct.name`, `imageUrls[0]` | real name, real image; placeholder graphic when absent — never a stock fake |
| Price + currency | effective `BusinessPrice` GHS row (exact 2dp; companion doc §1) | merchant-confirmed GHS list price; string-serialized 2dp; items without a confirmed GHS price never render a converted/estimated/fabricated figure |
| Business name + direct link | `BusinessProfile.businessName` | taps through to existing business profile flow (§5.1) |
| Availability | `isAvailable` + section window + branch | hidden when unsupported, e.g. no fake "sold out" states |
| Location/branch | `locationId` → nearest active branch label | only when location data exists |
| Prep/delivery indicators | `preparationMins`, `deliveryTerms`, `estimatedDelivery` | only where source data supports them |
| Popularity label | server-computed from real events (§5) | e.g. "Often ordered with Azaman" only when backed by measured counts; thresholds defined in §5.2 |

**Removed from default Mix ranking:** numerical stars, `averageRating` sorting (`world_deck.dart:70`), Top-Rated ordering and the Wilson `discoveryScore` (backend `utils/ranking.js`). Reviews/ratings remain visible inside a business's own page until separately reviewed. **Do not manufacture** avatars, order counts as social proof, or fake "popular" tags — the existing `tags: ["POPULAR"]` is owner-declared and must NOT be used as a popularity label in Mix.

### 2.3 Shop, Stay, Ride, services — shared infrastructure, distinct models

All verticals share: the discovery API (§3), the item-event contract (§5), eligibility filtering, pagination, caching, and the Mix/Stores toggle. They differ in **action capability** and **unit semantics**, expressed by `action` + `capability` flags in the DTO, not by food-specific fields:

- **Shop (RETAIL):** stock-aware (`stockQty`, `sku`); `action: ADD_TO_CART`; variants/modifiers.
- **Stay (HOSPITALITY):** unit is a room/offer; `action: CHECK_AVAILABILITY` → existing reservation flow (`POST /marketplace/business/:bizId/reservations`); date-window eligibility is NEW metadata, surfaced only when the business has bookable inventory.
- **Ride (LOGISTICS):** unit is a trip; `action: VIEW_SEATS` → existing transit flow (`/marketplace/transit/...`); departure-time windows replace prep-time.
- **Services (OTHER):** `action` reflects the business's supported channels (invoice, DM, booking) — never a generic "Buy".

The generalization rule: every vertical-specific field is optional in the DTO, and the client hides what is absent. No vertical is forced to fill food fields.

### 2.4 Flows

1. Home → Marketplace (Mix, last-used category) → filtered feed → item card → item detail (NEW lightweight sheet: full description, variants/modifiers, business row) → business handoff or add-to-cart (single business).
2. Item card → business row tap → **Stores mode on that business** with category/search context preserved; back returns to Mix at the same scroll anchor.
3. Stores toggle → existing business rails/list, unchanged.

---

## 3. Cross-business discovery API contract

### 3.1 Endpoint

**NEW** `GET /api/discovery/items` (public, no auth; soft rate-limit by IP+session). Single bounded query — no client-side N+1.

Request:
```
GET /api/discovery/items
  ?mode=mix
  &category=FOOD_BEVERAGE          # business category enum (Eat/Shop/...)
  &taxonomy=MEALS                  # canonical filter (§4); repeatable
  &q=jollof                        # item-name/description search
  &lat=5.6037&lng=-0.1870&radiusKm=10
  &cursor=<opaque>&limit=24        # limit ≤ 50, default 24
  &session=<anon-id>              # optional, for For You ranking fallback only
```

Response (example):
```json
{
  "success": true,
  "items": [{
    "id": "9f1c...",                       // BusinessProduct.id
    "businessId": "3a7b...",               // BusinessProfile.id
    "bizId": "BIZ-123456789",
    "businessName": "Auntie Adjoa's Kitchen",
    "businessCategory": "FOOD_BEVERAGE",
    "item": {
      "name": "Jollof Rice with Chicken",
      "slug": "jollof-rice-...",
      "description": "...",
      "imageUrl": "https://res.cloudinary.com/...",
      "images": ["..."],
      "tags": ["SPICY"]
    },
    "pricing": {
      "amount": "25.00",                   // exact 2dp GHS string from the effective BusinessPrice row
      "currency": "GHS",                   // canonical listing currency for Ghana (D-1, corrected §3.4)
      "displayCurrency": "GHS",
      "hasUsdcListPrice": true             // informational only; never used to price or estimate
    },
    "variants": [{ "id": "v_small", "label": "Small", "priceDeltaGhs": "0" },
                 { "id": "v_large", "label": "Large", "priceDeltaGhs": "5.00" }],
    "availability": {
      "isAvailable": true,                 // product toggle
      "sectionWindowOpen": true,           // CatalogSection time window at browse time (server clock)
      "preparationMins": 20,
      "deliveryTerms": "Same day delivery",
      "estimatedDelivery": null
    },
    "location": { "locationId": "loc_1", "label": "Osu branch", "distanceKm": 2.4 },
    "section": { "id": "sec_9", "name": "Main Course", "taxonomy": "MEALS" },
    "action": "ADD_TO_CART",               // ADD_TO_CART | BOOK | VIEW_SEATS | VIEW_MENU | ENQUIRE
    // Payment capability is reported separately from listing currency and only
    // from actually-supported rails (D-1 corrected + owner rule: browsing ≠ payable).
    // "GHS" appears here ONLY after the GHS collection rail ships (companion §3.3).
    "capabilities": { "orderable": true, "bookable": false, "deliverable": true, "dineIn": false,
                      "payableIn": ["USDC"] },
    "rank": { "score": 0.412, "label": "OFTEN_ORDERED" }  // §5.2; label null when unsupported
  }],
  "nextCursor": "eyJ...",
  "cache": { "maxAgeSec": 60 }
}
```

**Exclusions (server-enforced where clause):** `BusinessProduct.isActive=false` or `isAvailable=false`; section `isActive=false` or window closed at browse time; business `isSuspended=true` or `isPausedByOwner=true` (fixes B-1); **no business-confirmed effective GHS price row** (D-1 corrected — a missing GHS price never falls back to `priceUsdc`, an estimate, or any conversion; see companion doc §3.2/§4); items whose `locationId` names a branch inactive or outside the user's region filter; business not publicly discoverable (NEW marketplace-visibility flag — separate capability from the custom storefront, see §9 D-5 audit).
**Explicitly NOT a Mix gate:** a published custom storefront layout (D-5 decided — layout publication is not required for Mix eligibility) and `storefrontDisabled` (an admin kill-switch scoped to the custom SDUI storefront experience; audited in §9 D-5 — marketplace visibility and the custom storefront are modeled as separate capabilities, so Mix never bypasses the storefront opt-out and never inherits it).
**Stores "Price on request" (owner-decided):** quotation-based pricing is an explicit per-item business opt-in, never inferred from a missing price. Unpriced items are excluded from the customer-facing catalogue (Mix and Stores) until a confirmed GHS price exists, and surface in the portal worklist. **Public DTO never exposes** owner `userId`, emails, `costPrice`, `supplierId`, `totalRevenue`, or internal moderation fields.

### 3.2 Ordering, stability, cache

Stable ordering key: `(rankScore desc, businessDiversityRounds asc, BusinessProduct.id asc)` — cursor encodes the full key tuple (opaque base64, versioned) so pages are stable across cache refreshes; a product that changes price mid-browse keeps its position until the next first-page request. First-page responses cached (Redis, TTL 60s, key = normalized query) exactly like `storefrontRenderService`'s cache-with-invalidation pattern; invalidation hooks on product/section/business writes. Pagination is cursor-only — no offset scan amplification.

### 3.3 Indexes (NEW migration, additive)

```
CREATE INDEX business_product_discovery
  ON business_products (category, is_active, is_available);
CREATE INDEX business_product_section_active
  ON business_products (catalog_section_id, is_active);
CREATE INDEX business_profile_eligible
  ON business_profiles (category, is_suspended, is_paused_by_owner);
-- PostGIS or lat/lng bounding-box index deferred to D-6 (location filter v1 = city/region string).
```

### 3.4 Pricing truth for the Ghana market (decision D-1 — CORRECTED, owner-confirmed)

**The canonical displayed/listing price for Ghana Marketplace products and services is GHS.** USDC remains a supported payment/funding asset where checkout actually supports it, but it is not the default listing currency. The earlier D-1a USDC-default is withdrawn.

Contract (full detail in `MARKETPLACE_GHS_PRICING_AND_PAYMENT_CONTRACT_2026-10-10.md`):

- **Price storage (D-1 follow-up DECIDED, owner-approved):** a normalized, scoped, effective-dated `BusinessPrice` model (exact 2dp `Decimal(20,2)`, `currency`, `scope DEFAULT|BRANCH`, `confirmedByType BUSINESS|ADMIN|MIGRATION` + confirming actor and timestamp) — approved over an additive `priceGhs` column. Owner-added constraints (companion doc §1.2): database-enforced uniqueness for the single current active effective price per product/currency/scope (partial unique index), effective-date integrity (`effectiveTo` is null or strictly after `effectiveFrom`; no overlapping active windows), an auditable confirming actor + timestamp on every confirmation, and a positive base GHS price unless a separately approved free-item model permits zero. Variant/modifier deltas use exact-decimal parsing with bounds and final-price validation — never binary floating-point arithmetic.
- **Decimal truth:** GHS amounts are exact 2-fractional-digit decimals end to end (pesewas-integer discipline per the existing `paymentRequestService` / `toExactGhsDecimal` patterns). Never binary floating-point money, client or server.
- **Honesty rules preserved:** never relabel `priceUsdc` as GHS, never convert USDC into a GHS price, never display a USDC→GHS estimate as the primary user price, never silently fall back to the legacy USDC amount. A product without a business-confirmed GHS price is **excluded from Mix and from the customer-facing Stores catalogue** ("Price on request" only on explicit quotation-pricing opt-in — companion doc §4) and surfaces in a portal worklist — never shown with a fabricated or mislabelled price.
- **Payment reality (verified from code):** no GHS merchant-checkout collection path exists today. Moolre collections are a deposit rail into Azaman's own account (credits the USDC wallet); escrow checkout is USDC-only; order snapshots carry no currency field. GHS checkout is a **hard dependency** before any product copy claims customers can pay in GHS: its collection, callback, reconciliation, ledger and merchant-settlement work stays behind its own financial review (companion doc §3.3). No invented payment success.
- **Listing, payment and settlement are three separate facts (owner rule):** every GHS-listed order preserves the agreed GHS listing-price snapshot; every payment records the actual payment amount/currency (and, for USDC payment, the server-confirmed quote identity, source, timestamp and expiry); merchant payable amounts/currency and settlement references are separately represented. One `currency` field must never mean all three at once.
- **Merchant proceeds stay GHS-native by default:** the preferred local-market design is GHS merchant payable accounting with supported merchant payouts — an investigation workstream (companion doc §3.3.3), NOT an implementation assumption. Merchant proceeds must NOT be automatically converted to USDC without explicit owner approval and a separately reviewed financial contract.
- USDC payment, where offered, requires a server-authoritative quote (approved rate source, expiry, user confirmation) derived at order time — the GHS list price stays authoritative regardless of how the customer funds payment.

---

## 4. Taxonomy and availability contract

**Canonical taxonomy table (NEW `catalog_section_taxonomy`, D-2 DECIDED — mapping-table default + owner override approved):** `id` (MEALS, STARTERS, SIDES, DRINKS, DESSERTS, ROOMS, RIDES, RETAIL_ITEMS, SERVICES, UNMAPPED), display metadata, per-category validity.

**Mapping rule:** deterministic name-match first (normalized section name → canonical id, seeded with the common Ghana food-house names: "Main Course"→MEALS, "Starters/Appetizers"→STARTERS, "Drinks/Beverages"→DRINKS, "Sides/Extras"→SIDES, "Desserts/Sweets"→DESSERTS). Unmatched sections: `UNMAPPED` — items stay discoverable under the category's general feed but never under a specific chip; the business portal surfaces a one-tap "categorize this section" suggestion. **Mapping is a property of the section, owned by the mapping table + optional owner override field — never a silent client-side guess.**

**Availability truth at query time:** product `isAvailable` AND section `isActive` AND (section window open at server time) AND business eligible. The server computes `sectionWindowOpen` once per request batch (group by section), not per item. Timezone: Africa/Accra, single zone for v1 (all hours fields are local-time strings today).

**Prep/delivery fields** (`preparationMins`, `deliveryTerms`, `estimatedDelivery`) pass through verbatim when present; the client renders them only when non-null. **No invented delivery promises.**

---

## 5. Analytics event schema and ranking baseline

### 5.1 Item-level interaction contract (NEW `MarketplaceDiscoveryEvent` table — D-3 DECIDED: separate from `StorefrontAnalyticsEvent`, approved)

One row per event: `id`, `eventType`, `productId` (nullable), `businessProfileId`, `sectionId?`, `searchQueryId?` (the discovery query hash that produced the impression), `surface` (MIX_FEED, MIX_DETAIL, STORES), `visitorId` (anon cookie/device id), `userId?` (authenticated), `sessionId`, `clientEventId` (uuid v4, client-generated for dedup), `occurredAt` (client clock, informational), `receivedAt` (server clock, authoritative), `metadata Json`. Unique index on `(clientEventId)` — replays are idempotent inserts-once.

**Event types and trust rules:**
- `ITEM_IMPRESSION` — client fires only when ≥50% of the card is visible ≥1s in the viewport (IntersectionObserver-equivalent), max one per card per feed instance per session; feed rebuilds re-show the same card within the same session **do not** re-fire (client keeps an in-session shown-set).
- `ITEM_TAP` (detail open), `SEARCH_SUBMIT` (query + filters), `FILTER_APPLY` (taxonomy chips), `SAVE_ITEM`/`UNSAVE_ITEM`, `VARIANT_SELECT`, `ADD_TO_CART` (with price snapshot + business id), `CHECKOUT_START`, `ORDER_CONFIRMED`, `BOOKING_CONFIRMED`, `ORDER_CANCELLED`, `REFUND_ISSUED`.
- **Client events can never fabricate outcomes:** `ORDER_CONFIRMED`/`BOOKING_CONFIRMED`/`REFUND_ISSUED` are written **only** by the backend inside the existing order/reservation/refund success paths (server-derived, referencing the real order id), not accepted from client payloads. Client sends everything else in **batched** flushes (≤25 events or 10s, `POST /api/discovery/events:batch`, fire-and-forget queue, non-blocking UI, dropped-on-failure without retry storm; single retry with backoff max 1).

**Privacy:** visitor id is a random client-generated token, no PII; `userId` only when authenticated; metadata schema allowlisted; events excluded for opted-out sessions; owner-facing aggregates only (never per-visitor drilldown in the business portal).

### 5.2 First-stage ranking (testable baseline — no ML)

Score per item for a given request context:

```
base   = w1 * log1p(confirmed_orders_28d_item)      // real outcomes only (ORDER_CONFIRMED)
       + w2 * log1p(distinct_buyers_28d_business)   // business momentum
       + w3 * freshness(created_at decay 14d)       // fair exposure for new items
       + w4 * open_now_bonus                        // availability at browse time
       + w5 * proximity(distanceKm, capped)         // when location context given
       + w6 * personal_affinity(item, user history) // For You only; falls back to w7
       + w7 * default_interest(category, region popularity)
```

- **Proposed starting weights** (labeled proposed until calibrated §8): w1=0.35, w2=0.15, w3=0.10, w4=0.10, w5=0.15, w6=0.10, w7=0.05. Calibration = offline replay against logged events + A/B exposure at rollout, not fixed opinions.
- **Diversity guard (D-8 DECIDED — configurable initial defaults, not code constants):** per page of 24, max 6 items from one business (round-robin interleave after first-pass sort) and fair-exposure ε=0.10 of page slots for items below a 28-day impression floor. Both live in server configuration. **Cap relaxation rule:** when the eligible pool has insufficient alternatives to fill the page under the cap, the cap relaxes to the minimum extent needed to complete the page from the next-best *eligible* items — never by admitting ineligible ones; the applied cap is returned in the response for observability. **Calibration (owner rule):** weights, cap and ε are recalibrated from actual impressions, taps, cart additions and server-confirmed outcomes (offline replay against logged events + A/B exposure at rollout) — never from guesswork; pre-calibration values are labeled proposed in telemetry.
- **Surface separation (D-4 DECIDED):** Stores and Mix use separate ranking logic. Stores keeps the business-first contract (business-level ordering unchanged); the item ranker applies ONLY to the Mix feed and never silently replaces Stores ordering. Rating-driven discovery is removed from Mix (numerical stars, `averageRating` sorting, Wilson `discoveryScore` — B-2); reviews remain visible inside a business's own page.
- **Honest labels:** `POPULAR_NEARBY` = top decile confirmed orders within radius in 28d (min 10 orders); `OFTEN_ORDERED` = top quartile category-wide (min 25 orders). Both computed from the aggregate table below; if thresholds aren't met, `label: null`. No stars, no volume claims beyond these.
- **Abuse resistance:** event ingestion rate-limits per visitor/session; outcome events are server-authoritative (5.1); rank inputs come only from server-derived aggregates (`product_engagement_daily` rollup table: productId, day, impressions, taps, carts, orders — built by a scheduled rollup, not by reading raw events at query time).
- **Fallback for new users:** w7 path — category-wide engagement ordering; identical plumbing, no special-casing in the client.

### 5.3 Preserving the individual business experience + future multi-merchant

Mix → business handoff passes `(businessId, productId, variantSelection)` into the existing per-business flow: business profile screen → its menu → its cart (single-business cart is unchanged — the existing `CartState` clears on business switch, and **products from multiple businesses must never silently share one order**). The item detail sheet's add-to-cart uses the same single-business cart with the current business-clearing semantics made explicit in UI copy ("Starting a cart with Auntie Adjoa's Kitchen").

**Future architecture (documented only, not built):** multi-merchant checkout = a *basket of independent single-business orders*: one payment intent → N escrows, per-business fulfilment, fees, settlement and refund lifecycles; requires a coordination record above `BusinessOrder`, settlement split rules, and a refund-per-merchant policy. This must go through its own financial review (FINANCIAL_INVARIANTS.md) — explicitly out of scope for this phase.

**Capability-driven actions:** the DTO's `action`+`capabilities` (§3.1) are computed server-side from what the business actually supports (has products + not paused → orderable; has availability rules/reservation support → bookable; transit trips → VIEW_SEATS; otherwise ENQUIRE pointing at invoice/DM). **A business never sees a delivery, booking, or payment button it does not support.**

---

## 6. Public business pages for web search

**Greenfield.** No SSR, sitemap, slugs, or public web surface exists (§1.2). Recommended architecture:

- **Surface (D-7 DECIDED — canonical public business pages approved):** a small server-rendered (or build-time prerendered) public web app served by the backend's existing Express host on a dedicated hostname — crawlable SSR/prerendered HTML, stable slugs, redirects, canonical metadata, structured data and sitemaps are all in scope. **The hostname remains deployment configuration until the owner/domain authority verifies the production domain — no speculative domain is hard-coded anywhere**; URLs in this document are placeholders of the form `https://<production-domain>/business/<slug>`.
- **Slugs (NEW `business_slug` table):** `id`, `businessProfileId`, `slug` (unique, generated from business name with collision suffix), `isCanonical` + `redirectsFrom` history. Renames create a new canonical slug and a 301 from the old one — stable URLs forever. Owner-editable with normalization + reserved-word list.
- **Requirements:** real `<title>`/meta description (business name + category + city), OpenGraph/Twitter cards with cover image, `rel=canonical`, JSON-LD (`LocalBusiness`/`Restaurant`/`Store` + `Menu`/`ProductList` where published), `robots.txt` + sitemap index (per-page sitemaps ≤50k URLs, regenerated on publish events), catalogue/menu content rendered as crawlable HTML from the business's **active product data** — publication of a custom storefront layout is NOT required (D-5 decided); businesses without an active public catalogue get a minimal profile page.
- **Flutter deep links:** add universal/app links (`https://<production-domain>/business/<slug>?src=web`, domain from deployment config once verified) alongside the existing `azaman://` scheme (`deep_link.dart` rewrites both); the web page is the fallback itself — the "use the app" interstitial never blocks crawlers.
- **Owner control:** a per-business `publicPageEnabled` toggle (default on for published storefronts), suppress-list for specific fields, and a removal path: unpublish → deindex (page returns 410 + `X-Robots-Tag: noindex`, removed from sitemap) — moderation can force-unpublish suspended businesses' pages immediately (mirrors the `isSuspended` marketplace rule).
- **SEO privacy/moderation:** pages expose only public fields already in the marketplace DTO (§3.1 exclusions); phone/address only when the business already shows them in-app; review content screened by existing moderation before rendering; noindex for suspended/paused.

**Honesty:** this implementation controls crawlability, canonical URLs, structured data, and sitemap freshness. **It cannot promise rankings** — those depend on Google's opaque signals (link authority, engagement) outside AZM's control.

---

## 7. Data migration and backward-compatibility

All changes additive; no destructive migration:

1. `catalog_section_taxonomy` + section taxonomy id column (default UNMAPPED) — backfill job maps by name; runs idempotently.
2. `business_slug` table + backfill from business names.
3. `MarketplaceDiscoveryEvent` + `product_engagement_daily` + batch endpoint — no existing table touched.
4. Discovery indexes (§3.3) — online CREATE INDEX, no lock rewrite.
5. NEW endpoints under `/api/discovery/*` — existing `/api/business/search`, storefront routes unchanged (B-1/B-3 fixes are separate small PRs with their own tests, listed in §9 blockers).
6. Frontend: Mix is additive screens/providers; `MerchantCard`/Stores mode untouched; `world_deck.dart:70` star sort removal is a one-line change shipped inside the Mix PR with updated widget tests.
7. **Sequencing with open PRs:** #147 touches `app_router.dart`/`route_registry.dart`; the Mix PR adds routes for marketplace sub-paths — rebase Mix after #147 lands (no logic overlap). #332 is backend-only additive; no interaction. Merge order: #320/#317 (existing queue) → #332/#147 → Mix PRs.
8. **GHS pricing (D-1 corrected, follow-ups DECIDED):** `BusinessPrice` table with database-enforced constraints (partial unique index: one ACTIVE row per product/currency/scope/locationId; effective-date integrity: `effectiveTo` null or strictly after `effectiveFrom`; confirming actor + timestamp columns) + variants/modifierGroups `priceDeltaGhs` JSON option + nullable listing/payment/settlement fields on order models (companion doc §5 phase P1); source-backed backfill drafts ONLY (never derived from `priceRange` or converted from `priceUsdc`) + portal confirm worklist (P2); discovery/Mix GHS pricing (P3); order-snapshot currency separation + checkout GHS revalidation (P4). All additive; legacy USDC rows and read paths unchanged.
9. **GHS collection rail (companion doc §3.3, P6):** separate PR, own financial review — GHS checkout is a hard dependency before any GHS-payment claim. Blocked on: merchant GHS payable accounting investigation (preferred local-market design), order-quote rate source approval, rail scheduling. Not part of Mix phases P1-P5.
10. **Capability separation (D-5 audit outcome):** NEW explicit marketplace-visibility flag on `BusinessProfile` (public discoverability), kept separate from `storefrontDisabled` (admin kill-switch scoped to the custom SDUI storefront experience — render, theme, storefront delivery, checkout readiness, safe-publish). Both are honored on their own surfaces; neither bypasses the other. Also NEW: explicit quotation-pricing opt-in (per-item) for the Stores "Price on request" state.
11. **Location filter (D-6 DECIDED):** city/region string filters for v1; radius queries deferred until accurate, indexed geo queries are verified as suitable (no PostGIS migration in this phase).

**Never in this phase:** changing `priceUsdc`/`amountUsdc` semantics or values, the single-business cart, or the escrow flow. GHS is additive data with its own fields, validation and audit path.

---

## 8. Test strategy, performance measurement, and rollout phases

**Tests:** (a) contract tests pinning the discovery DTO (field presence, exclusion matrix: suspended/paused/unavailable/unpublished/region-ineligible — one PG test per exclusion); (b) taxonomy mapping unit tests incl. rename + window-closed cases; (c) analytics ingestion tests: batch idempotency on `clientEventId`, client-forged `ORDER_CONFIRMED` rejected (403 path), rate-limit behavior; (d) ranking tests: diversity cap (no business >6/24), **cap relaxation when the eligible pool has insufficient alternatives (page fills only from eligible items; applied cap observable)**, fair-exposure slot presence, label thresholds (min counts), fallback ordering determinism, **config-driven cap/ε (changed config → changed behavior, no code constant)**; (e) cursor stability test: same key tuple across cache refresh; (f) frontend widget tests: Mix feed renders honest card states (missing image, no prep time, no label), mode-switch context retention, business handoff and back-to-anchor; (g) SSR page tests: slug redirect 301, 410 on deindex, JSON-LD validity, noindex on suspended;
(h) GHS price-constraint tests: second ACTIVE price row per product/currency/scope rejected by the DB unique index; `effectiveTo ≤ effectiveFrom` rejected; non-positive base price rejected (until a free-item model is separately approved); `priceDeltaGhs` exact-decimal parsing, bounds and final-price validation (no float arithmetic); MIGRATION-drafted rows never confer Mix eligibility;
(i) currency-separation tests: GHS listing snapshot, payment record (amount/currency + USDC quote identity/source/timestamp/expiry when applicable) and merchant payable fields are distinct columns on the order path — mutating one never alters the others; `payableIn` in the public DTO reflects only actually-supported rails (no "GHS" until the rail ships);
(j) capability-separation tests (D-5): `storefrontDisabled` hides the custom storefront experience but does not remove marketplace visibility (and vice versa for the marketplace-visibility flag); "Price on request" renders only on explicit quotation opt-in.

**Performance contract** — likely expensive paths and *proposed* acceptance criteria (labeled proposed until baselined on representative devices, Android mid-range; no fabricated numbers, no reduced-motion masking):

| Path | Expensive today | Proposed target (to be baselined first) |
|---|---|---|
| Home → Marketplace open | portal builds many rails eagerly | first frame < 400ms; Mix feed request fires before rail builds |
| Category/mode switch | re-search round trip | < 300ms perceived via cached first page (60s TTL) + optimistic chips |
| First + next pages | cursor query | p95 server < 150ms (bounded take, §3.3 indexes, no N+1 — single query + 1 batch availability computation) |
| Card scrolling/images | full-res Cloudinary URLs | sized/cached transforms (`f_auto,w_600`), placeholder + fade; scroll jank measured via trace, no rebuild storms (item cards `RepaintBoundary` + keys like MerchantCard) |
| Business open & return | full profile fetch | existing handoff + scroll-anchor restore < 500ms |
| Analytics | per-request inserts | client batch (10s/25 events), fire-and-forget; zero UI-thread cost measured |

Measurement plan: baseline traces on main BEFORE Mix lands (DevTools timeline + server p95 from logs), same traces after; improvement = measured deltas on the same device class, recorded in the PR. Explicitly rejected: "feels faster" claims without traces.

**Rollout phases:** P1 backend discovery API + taxonomy + indexes (feature-flagged, no UI); P2 Mix feed UI (Eat only) behind a client flag with A/B exposure for ranking calibration; P3 analytics batch + ranking labels (server-side only); P4 Shop/Stay/Ride/service actions; P5 public web pages + slugs + sitemap; P6 mobile web-ordering surface reusing `public-theme`/`render`. Each phase = one PR, full CI battery (backend test + financial-durability; frontend FQ + Android Integration), route-check, analyzer clean, dead code removed before review.

---

## 9. Risks, blockers, assumptions, decisions needing approval

**Audit findings to fix regardless (small standalone PRs, blockers for Mix correctness):**
- B-1: `searchBusinesses` ignores `isPausedByOwner` (`services/businessService.js:124` where-clause) — paused businesses are discoverable today (order attempt is correctly rejected).
- B-2: default sort `totalEscrows desc` + `trending` Wilson-star sort must be replaced by the Mix ranker; store-mode may keep totalEscrows ordering (decision D-4).
- B-3: `/api/storefront/discover` category filter is structurally broken (`routes/storefrontRoutes.js:106` nested `businessProfile.businessProfile`).
- B-4: `tags: ["POPULAR","NEW"]` are owner-declared; Mix must not surface them as evidence-backed labels.

**Decisions (all finalized 2026-10-10 on head 7c813fb — no decision remains open in this document):**
- **D-1 price display: DECIDED.** GHS is the canonical listing currency for Ghana Marketplace; Mix shows business-confirmed GHS prices, excludes products without one, never falls back to `priceUsdc` or an estimate (§3.4, companion contract). All D-1 pricing follow-ups decided: normalized `BusinessPrice` approved with DB-enforced uniqueness, effective-date integrity, auditable confirming actor + timestamp; source-backed backfill drafts only (never from `priceRange`, never converted from `priceUsdc`); positive base price unless a separately approved free-item model permits zero; exact-decimal variant/modifier deltas with bounds and final-price validation; listing/payment/settlement recorded as three separate facts.
- **D-2 taxonomy authority: DECIDED.** Canonical section taxonomy with mapping-table default + owner override approved (§4).
- **D-3 new-table strategy: DECIDED.** Separate `MarketplaceDiscoveryEvent` model approved — different grain and trust model from `StorefrontAnalyticsEvent` (§5.1).
- **D-4 Stores-mode ranking: DECIDED.** Stores and Mix use separate ranking logic. Stores keeps the business-first contract; rating-driven discovery is removed from Mix and the item recommendation score never silently replaces Stores ordering (§5.2).
- **D-5 storefront-publishation gate: DECIDED.** A published custom storefront layout is NOT required for Mix eligibility. Eligibility = publicly discoverable business, active/unpaused/unsuspended business, active/available product, valid section/time window, branch/location rules, confirmed GHS price. **`storefrontDisabled` audit (verified from code):** the flag is written only by admin routes (`adminStorefrontRoutes.js:39/53`) and read only by the custom-storefront path — `storefrontRenderService.js:132` (published layout render), `storefrontSafePublishService.js:67` (safe publish), `storefrontCheckoutReadinessRoutes.js:41/57` (checkout readiness + public delivery of render/products/theme/public-theme) — while business search, `/api/storefront/discover` and the legacy single-item order endpoint never consult it. It therefore does NOT today conflate the custom storefront with marketplace visibility; it is an experience-scoped admin kill-switch. Per the owner rule, the two capabilities are modeled explicitly and separately (§7 step 10): marketplace visibility gains its own flag, `storefrontDisabled` keeps its narrow semantics, and neither bypasses the other. Latent inconsistency recorded for the implementation phases: the modern configured checkout is readiness-gated but the legacy single-item order endpoint is not.
- **D-6 location filter v1: DECIDED.** City/region string filters; radius queries deferred until accurate, indexed geo queries are verified as suitable (§7 step 11).
- **D-7 public web host & domain: DECIDED.** Canonical public business pages approved: crawlable SSR/pre-rendered HTML, stable slugs, 301 redirects, canonical metadata, structured data (JSON-LD), sitemaps, served from the backend's Express host. The hostname remains deployment configuration until the owner/domain authority verifies the production domain — nothing may hard-code a speculative domain (§6).
- **D-8 diversity cap and fair exposure: DECIDED.** Max 6 items/business per 24-item page and ε=0.10 exploration are configurable initial defaults; the cap relaxes when the eligible pool has insufficient alternatives; calibration is from actual impressions, taps, cart additions and server-confirmed outcomes — not guesswork (§5.2).

**Remaining genuine blockers (explicit, none hidden in this document):**
1. **Merchant GHS payable contract:** investigation DELIVERED as `MERCHANT_GHS_PAYABLE_FINANCIAL_CONTRACT_2026-10-10.md` (business-scoped GHS payable, r15-based payout rail, GHS fee model, payable-first refunds, nine durability pins, §2 currency-invariant scope amendment, seven owner decision points) — awaiting owner sign-off before the GHS checkout rail's settlement code can be reviewed. Merchant proceeds are NOT automatically converted to USDC without explicit approval and a separately reviewed financial contract.
2. **GHS collection rail:** hard dependency for any "pay in GHS" claim; its collection, callback, reconciliation, ledger and merchant-settlement work needs its own financial review and a schedule.
3. **Order-quote rate source:** which rate source(s) may back USDC payment quotes for orders still needs explicit approval (deposit-quote authority exists; an order purpose must be configured explicitly).
4. **Production domain verification** for public business pages (deployment configuration, owner/domain authority).
5. **Free-item (zero-price) model:** needs separate approval before any zero-price listing is permitted.
6. **B-1/B-3 fix PRs:** addressed by open PR #333 (`fix/discovery-eligibility`, awaiting owner merge); B-2/B-4 land with the Mix PRs. Merge order queue from §7 step 7 still applies.
7. **Public web pages cannot promise search rankings** — crawlability, canonical URLs, structured data and sitemap freshness are controlled; rankings are not (honesty note, not a blocker).

**Assumptions (verify at implementation):** Africa/Accra single timezone; Cloudinary transformations available on existing URLs; Redis available in production (memory fallback exists); outcome events derivable from existing order/reservation/refund success paths without touching financial code paths beyond an emit hook.

**Risks:** event volume growth (mitigated: batching + rollup table, raw events pruned after 90d); taxonomy mapping backlog for long-tail section names (mitigated: UNMAPPED stays visible in the general feed); SSR surface expanding attack area (mitigated: public-fields-only DTO reuse, rate limits, cached renders); ranking cold-start labels showing nothing (honest by design).

**Recommended first implementation PRs (in order):**
1. `fix/discovery-eligibility` — B-1 + B-3 (+ tests), backend-only.
2. `feat/discovery-api` — §3 endpoint + §4 taxonomy + indexes, feature-flagged (P1).
3. `feat/mix-eat` — frontend Mix feed + item card + mode toggle (P2).
4. `feat/discovery-analytics` — §5 events + ranker + labels (P3).

— End of Phase 0 deliverable. No application source code was modified. Exact head and files for this documentation branch are reported in the accompanying session report.