# AZM Marketplace Mix Redesign — Architecture & Product Contract

> **Agent A, Phase 0 (documentation-only).** Verified against live main on 2026-10-10:
> frontend `9640face067f45d0cfda6bb95f1569af6878add7`, backend `052370518133c91bdfb119ff271782045194f180`.
> Open PRs #147 (frontend, receive-request-recent-interaction, head `3a656d8`) and #332 (backend, payment-requests-standalone, head `b51fa80`) were inspected for scope only; neither is modified by this phase.
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

**Currency:** every product/order/escrow amount is USDC (`priceUsdc`, `amountUsdc`). GHS appears only in conversion flows (`prisma/schema.prisma:1110` `currency` default GHS; `:1427` live retail rate for the USDC→GHS sell hologram; `GET /api/azm-conversion/rate` is authenticated). **There is no product-level GHS price and no public rate endpoint — see §3.4, decision D-1.**

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
| Price + currency | `BusinessProduct.priceUsdc` (+ §3.4 decision) | authoritative value as stored; never relabel USDC as GHS |
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
      "amount": "25.00",
      "currency": "USDC",                  // authoritative, as stored
      "displayCurrency": "USDC"            // see D-1; "USDC" until a GHS contract is approved
    },
    "variants": [{ "id": "v_small", "label": "Small", "priceDelta": "0" },
                 { "id": "v_large", "label": "Large", "priceDelta": "5.00" }],
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
    "capabilities": { "orderable": true, "bookable": false, "deliverable": true, "dineIn": false },
    "rank": { "score": 0.412, "label": "OFTEN_ORDERED" }  // §5.2; label null when unsupported
  }],
  "nextCursor": "eyJ...",
  "cache": { "maxAgeSec": 60 }
}
```

**Exclusions (server-enforced where clause):** `BusinessProduct.isActive=false` or `isAvailable=false`; section `isActive=false` or window closed at browse time; business `isSuspended=true` or `isPausedByOwner=true` (fixes B-1); catalog unpublished for the storefront-publishing case (per D-5); items whose `locationId` names a branch inactive or outside the user's region filter. **Public DTO never exposes** owner `userId`, emails, `costPrice`, `supplierId`, `totalRevenue`, or internal moderation fields.

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

### 3.4 Pricing truth for the Ghana market (decision D-1)

Today a product's only authoritative price is USDC. **It would be dishonest to** display `priceUsdc` as GHS, invent an exchange rate client-side, or imply a GHS checkout exists. Options:

- **D-1a (default in this plan):** display the authoritative USDC price, labeled USDC, everywhere in Mix. Add a *clearly-labeled estimate* line ("≈ GH₵ 312 at the current sell rate") **only if** a public, cached, server-side rate endpoint is added (the existing conversion rate is authenticated and tied to trading semantics — it is NOT a marketplace display rate; do not reuse it).
- **D-1b (deferred):** business-owned GHS list prices with settlement still in USDC — requires schema, portal, and accounting decisions (§9) and must not be smuggled into Mix v1.

Until D-1 is decided: **USDC only.** This is the truthful baseline for price *comparison* (relative ordering is unaffected).

---

## 4. Taxonomy and availability contract

**Canonical taxonomy table (NEW `catalog_section_taxonomy`):** `id` (MEALS, STARTERS, SIDES, DRINKS, DESSERTS, ROOMS, RIDES, RETAIL_ITEMS, SERVICES, UNMAPPED), display metadata, per-category validity.

**Mapping rule:** deterministic name-match first (normalized section name → canonical id, seeded with the common Ghana food-house names: "Main Course"→MEALS, "Starters/Appetizers"→STARTERS, "Drinks/Beverages"→DRINKS, "Sides/Extras"→SIDES, "Desserts/Sweets"→DESSERTS). Unmatched sections: `UNMAPPED` — items stay discoverable under the category's general feed but never under a specific chip; the business portal surfaces a one-tap "categorize this section" suggestion. **Mapping is a property of the section, owned by the mapping table + optional owner override field — never a silent client-side guess.**

**Availability truth at query time:** product `isAvailable` AND section `isActive` AND (section window open at server time) AND business eligible. The server computes `sectionWindowOpen` once per request batch (group by section), not per item. Timezone: Africa/Accra, single zone for v1 (all hours fields are local-time strings today).

**Prep/delivery fields** (`preparationMins`, `deliveryTerms`, `estimatedDelivery`) pass through verbatim when present; the client renders them only when non-null. **No invented delivery promises.**

---

## 5. Analytics event schema and ranking baseline

### 5.1 Item-level interaction contract (NEW `MarketplaceDiscoveryEvent` table)

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
- **Diversity guard:** per page of 24, max 6 items from one business (round-robin interleave after first-pass sort). **Fair exposure:** items below a 28-day impression floor get a bounded exploration slot (ε=0.10 of page slots) — new/low-data businesses appear without dominating.
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

- **Surface:** a small server-rendered (or build-time prerendered) public web app served by the backend's existing Express host on a dedicated hostname (e.g. `azaman.com/business/<slug>`), rendering `BusinessProfile` + published catalogue/menu from cached JSON (reuse the `storefrontRenderService` cache pattern; SSR via lightweight template rendering, not the React portals).
- **Slugs (NEW `business_slug` table):** `id`, `businessProfileId`, `slug` (unique, generated from business name with collision suffix), `isCanonical` + `redirectsFrom` history. Renames create a new canonical slug and a 301 from the old one — stable URLs forever. Owner-editable with normalization + reserved-word list.
- **Requirements:** real `<title>`/meta description (business name + category + city), OpenGraph/Twitter cards with cover image, `rel=canonical`, JSON-LD (`LocalBusiness`/`Restaurant`/`Store` + `Menu`/`ProductList` where published), `robots.txt` + sitemap index (per-page sitemaps ≤50k URLs, regenerated on publish events), published catalogue/menu content rendered as crawlable HTML **only when the business has a published storefront layout** (unpublished businesses get a minimal profile page).
- **Flutter deep links:** add universal/app links (`https://azaman.com/business/<slug>?src=web`) alongside the existing `azaman://` scheme (`deep_link.dart` rewrites both); the web page is the fallback itself — the "use the app" interstitial never blocks crawlers.
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

**Never in this phase:** changing `priceUsdc` semantics, the single-business cart, or the escrow flow.

---

## 8. Test strategy, performance measurement, and rollout phases

**Tests:** (a) contract tests pinning the discovery DTO (field presence, exclusion matrix: suspended/paused/unavailable/unpublished/region-ineligible — one PG test per exclusion); (b) taxonomy mapping unit tests incl. rename + window-closed cases; (c) analytics ingestion tests: batch idempotency on `clientEventId`, client-forged `ORDER_CONFIRMED` rejected (403 path), rate-limit behavior; (d) ranking tests: diversity cap (no business >6/24), fair-exposure slot presence, label thresholds (min counts), fallback ordering determinism; (e) cursor stability test: same key tuple across cache refresh; (f) frontend widget tests: Mix feed renders honest card states (missing image, no prep time, no label), mode-switch context retention, business handoff and back-to-anchor; (g) SSR page tests: slug redirect 301, 410 on deindex, JSON-LD validity, noindex on suspended.

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

**Decisions needing owner approval:**
- **D-1 price display:** USDC-only truth in Mix v1 (recommended) vs public estimate line vs GHS list prices (§3.4). Blocks: any price-comparison marketing language.
- **D-2 taxonomy authority:** mapping-table default + owner override (recommended) vs owner-mandatory categorization.
- **D-3 new-table strategy:** `MarketplaceDiscoveryEvent` separate from `StorefrontAnalyticsEvent` (recommended — different grain and trust model) vs extending the existing table.
- **D-4 Stores-mode ranking:** keep business-level `totalEscrows` sort in Stores (recommended) vs applying Mix ranker everywhere.
- **D-5 storefront-publishation gate:** does Mix require a *published storefront layout* (narrower, consistent with `/storefront/discover`) or mere product existence (recommended for reach: active products + eligible business; publication gates only the public web page)?
- **D-6 location filter v1:** region/city strings (recommended, no schema change) vs PostGIS migration (defer).
- **D-7 public web host & domain:** which domain, and SSR inside the backend host (recommended) vs separate static prerender service.
- **D-8 diversity cap value (6/24) and fair-exposure ε (0.10):** proposed; owner sign-off or A/B calibration first.

**Assumptions (verify at implementation):** Africa/Accra single timezone; Cloudinary transformations available on existing URLs; Redis available in production (memory fallback exists); outcome events derivable from existing order/reservation/refund success paths without touching financial code paths beyond an emit hook.

**Risks:** event volume growth (mitigated: batching + rollup table, raw events pruned after 90d); taxonomy mapping backlog for long-tail section names (mitigated: UNMAPPED stays visible in the general feed); SSR surface expanding attack area (mitigated: public-fields-only DTO reuse, rate limits, cached renders); ranking cold-start labels showing nothing (honest by design).

**Recommended first implementation PRs (in order):**
1. `fix/discovery-eligibility` — B-1 + B-3 (+ tests), backend-only.
2. `feat/discovery-api` — §3 endpoint + §4 taxonomy + indexes, feature-flagged (P1).
3. `feat/mix-eat` — frontend Mix feed + item card + mode toggle (P2).
4. `feat/discovery-analytics` — §5 events + ranker + labels (P3).

— End of Phase 0 deliverable. No application source code was modified. Exact head and files for this documentation branch are reported in the accompanying session report.