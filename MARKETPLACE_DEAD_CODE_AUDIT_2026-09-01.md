# Marketplace Dead-Code Audit — 2026-09-01

## Status

**Phase:** inventory and reachability evidence collection

**Deletion policy:** no source deletion in this audit PR. Any removal requires a later, separately reviewed implementation PR after the evidence is complete.

**Baseline supplied by engineering handoff:** 187 Prisma models and 148 Flutter screens are in the marketplace/product surface inventory. Those totals are treated as audit scope, not as proof that any item is dead.

## Decision classes

- **CONFIRMED REACHABLE** — direct route/import/consumer evidence exists.
- **INDIRECTLY REACHABLE** — reached through a parent screen/provider/service rather than a top-level route.
- **CANDIDATE — NEEDS FINAL PROOF** — no active route/import evidence found in the current search pass, but dynamic navigation, deep links, generated references, or external consumers still need to be ruled out.
- **DO NOT DELETE** — evidence shows active use or authoritative backend coupling.

## Verified Flutter route graph

The current `lib/router/app_router.dart` contains a substantial business-market route graph, including:

| Screen / surface | Evidence | Classification |
|---|---|---|
| `MarketplaceHomeScreen` | `/business-market` route constructs `MarketplaceHomeScreen` | CONFIRMED REACHABLE |
| `BusinessProfileScreen` | `/business/:bizId` route constructs `BusinessProfileScreen`; it is the catch route after literal sub-routes | CONFIRMED REACHABLE |
| `BusinessSearchScreen` | `/business/search` route constructs `BusinessSearchScreen` | CONFIRMED REACHABLE |
| `SavedBusinessesScreen` | `/biz/saved` route constructs `SavedBusinessesScreen` | CONFIRMED REACHABLE |
| `BusinessRegisterScreen` | `/business/register` route constructs `BusinessRegisterScreen` | CONFIRMED REACHABLE |
| `BusinessNotificationsScreen` | `/business/notifications` route constructs `BusinessNotificationsScreen` | CONFIRMED REACHABLE |
| `BusinessProductsScreen` | `/business/:bizId/products` route constructs `BusinessProductsScreen` | CONFIRMED REACHABLE |
| `MyOrdersScreen` | `/business-market/orders` route constructs `MyOrdersScreen` | CONFIRMED REACHABLE |
| `MyInvoicesScreen` | `/business-market/invoices` route constructs `MyInvoicesScreen` | CONFIRMED REACHABLE |
| `InvoiceDetailScreen` | `/business-market/invoices/:invoiceId` route constructs `InvoiceDetailScreen` | CONFIRMED REACHABLE |
| `BusinessDashboardScreen` | `/business-market/dashboard` route constructs `BusinessDashboardScreen` | CONFIRMED REACHABLE |
| `CheckInQrScreen` | `/marketplace/booking/checkin-qr/:reservationId` route constructs `CheckInQrScreen` | CONFIRMED REACHABLE |
| `BusinessCheckInScreen` | `/marketplace/business/checkin` route constructs `BusinessCheckInScreen` | CONFIRMED REACHABLE |
| `TransitTripListScreen` | `/marketplace/transit` and `/business-market/:bizId/transit` routes | CONFIRMED REACHABLE |
| `TransitSeatSelectionScreen` | `/marketplace/transit/:tripId/seats` route | CONFIRMED REACHABLE |
| `HotelBookingScreen` | `/business-market/:bizId/hotel-booking` route | CONFIRMED REACHABLE |
| `DineInTabScreen` | `/business-market/dine-in/:tabId` route | CONFIRMED REACHABLE |
| `BusinessStoriesScreen` | `/business-market/:bizId/stories` route | CONFIRMED REACHABLE |

Source: `AZM-frontend/lib/router/app_router.dart` at `main` (commit `8ec3a63121d2b4ae707345582cf9376607beef89`).

## Verified indirect imports

Several apparently secondary marketplace files are definitely live because parent screens import them:

- `advanced_filter_sheet.dart` is imported by both `business_search_screen.dart` and `marketplace_home_screen.dart`.
- `business_reviews_section.dart` and `business_book_tab.dart` are imported by `business_profile_screen.dart`.
- `catalog_storefront_screen.dart` is imported by `business_profile_screen.dart`.

These are **DO NOT DELETE** in the current audit phase even if they have no route of their own.

## High-confidence removal candidate found

### `lib/screens/marketplace/bill_detail_screen.dart`

The screen is a complete `BillDetailScreen` implementation for finalized dine-in bills and calls `dineInTabProvider(...).payTab(...)`.

Current search evidence found:

- the file itself;
- an associated planning scratch reference (`scratch/markult_sections/section_37.txt`);
- **no production import/reference to `BillDetailScreen` in the current code-search pass**;
- it is **not** a `GoRoute` target in `lib/router/app_router.dart`.

Classification: **CANDIDATE — NEEDS FINAL PROOF**.

Before deletion, the follow-up audit must rule out:

1. dynamic string-based route navigation;
2. `showModalBottomSheet` / `Navigator.push` construction outside indexed references;
3. notification/deep-link actions that instantiate the class dynamically;
4. test-only, demo-mode, or generated references;
5. any backend endpoint/UI contract whose sole customer-facing consumer is this screen.

No deletion is proposed in this PR.

## Backend capability evidence

The Backend's `routes/businessRoutes.js` is an active, broad business surface mounted at `/api/business`. It currently exposes concrete routes for:

- business registration/profile/search;
- product catalog CRUD and public product lookup;
- business orders, business stats, delivery transition, and customer order history;
- business notifications;
- locations and tables;
- invoices including lookup/create/list/send/email/void/pay;
- reviews and review stats;
- public menu/catalog sections;
- vehicles and transit trips;
- reservations/check-in stats and recent check-ins;
- marketplace aggregate stats.

This confirms that the business marketplace is not a dead UI-only shell: the router and backend form a connected vertical with multiple live producers/consumers.

## Realtime consumer check from PR #72 contract change

### Flutter

`AZM-frontend/lib/services/socket_service.dart` registers:

- `escrow_funded`
- `escrow_settled`
- `escrow_pending_settlement`
- `escrow_disputed`
- `escrow_resolved`
- `escrow_terms_updated`
- `escrow_refunded`

The escrow handlers dispatch payloads as generic `Map<String, dynamic>` data. The ticket workspace forwards the map to the escrow provider. There is no dereference of `fundedAt` or `settledAt` in the inspected consumer path. The additive timestamp fields are therefore safely tolerated by the current Flutter consumer.

### Admin Portal

`AZM-adminPortal/src/hooks/useAdminRealtime.js` subscribes to `escrow_funded`, `escrow_settled`, `escrow_pending_settlement`, `escrow_refunded`, and related events through one invalidation handler. The handler treats escrow payloads as opaque objects and invalidates canonical queries; it does not read `fundedAt` or `settledAt`. The additive escrow timestamp fields therefore do not break the Admin realtime consumer.

## Current audit conclusions

1. The business marketplace contains many **confirmed reachable** screens; route count must not be used as a proxy for dead code.
2. Parent-child imports account for a meaningful amount of the screen/widget surface. Secondary files without routes can be fully live.
3. `BillDetailScreen` is the strongest removal candidate identified so far, but it remains **unconfirmed** until dynamic construction and hidden consumer paths are ruled out.
4. Backend route coverage is strong enough that frontend screens must be checked against concrete endpoint/service usage before classification.
5. No Prisma model is being deleted based solely on low visible usage. Model deletion requires producer/consumer/reference evidence and a separate migration-aware cleanup PR.

## Next audit pass

The next pass will systematically walk every marketplace screen and every marketplace Prisma model through this evidence chain:

`file → imports → router/dynamic navigation → provider/service → backend endpoint → Prisma producer/consumer → realtime/deep-link references`

The result will be a table of confirmed-live items, orphan candidates, superseded items, and unresolved cases with evidence links. Only after that review will a deletion PR be opened.
