# Marketplace Reachability Audit — Final Pass — 2026-09-01

This is the final evidence record following the documentation-only audit in PR #15 and the separate Frontend cleanup PR #38.

## Flutter marketplace file classification

### CONFIRMED REACHABLE

`marketplace_home_screen.dart`, `business_profile_screen.dart`, `business_search_screen.dart`, `saved_businesses_screen.dart`, `business_register_screen.dart`, `business_notifications_screen.dart`, `business_products_screen.dart`, `my_orders_screen.dart`, `my_invoices_screen.dart`, `invoice_detail_screen.dart`, `business_dashboard_screen.dart`, `checkin_qr_screen.dart`, `business_checkin_screen.dart`, `transit_trip_list_screen.dart`, `transit_seat_selection_screen.dart`, `hotel_booking_screen.dart`, `dinein_tab_screen.dart`, `business_stories_screen.dart`.

Evidence: each is a concrete target in `lib/router/app_router.dart`; the router also preserves the intended ordering for literal `/business/...` routes versus the `/:bizId` catch route.

### INDIRECTLY REACHABLE

`advanced_filter_sheet.dart` — imported by business search and marketplace home.

`business_book_tab.dart` — constructed by business profile.

`business_reviews_section.dart` — constructed by business profile.

`catalog_storefront_screen.dart` — imported by business profile.

`cart_screen.dart` — imported by `storefront_screen.dart` and part of the storefront purchase path.

`leave_review_sheet.dart` — constructed by `collapsible_business_bar.dart`, order tracking, and invoice detail flows.

`receipt_screen.dart` — constructed by invoice detail after payment.

These have no standalone GoRoute requirement; parent construction proves reachability.

### DEAD — PROVED AND REMOVED

`bill_detail_screen.dart` — no production constructor/reference, no GoRoute target, no route-token/string reference, no notification/deep-link construction, and no dynamic `BillDetailScreen(...)` instantiation outside the file. The underlying dine-in payment capability remains reachable through the active dine-in flow. Removed in Frontend PR #38.

`category_drilldown_screen.dart` — no production constructor/reference and no route/deep-link construction. The underlying `/business/subcategories` capability remains exposed by Backend and consumed by the active business search stack, but this particular screen is not part of the live navigation graph. Removed in Frontend PR #38.

## Backend marketplace capability graph

The active `/api/business` router provides concrete producer/consumer boundaries for:

- business registration/profile/search;
- products and product lookup;
- orders, stats, delivery transition, and customer order history;
- business notifications;
- locations/tables;
- invoices (lookup/create/list/send/email/void/pay);
- reviews and review statistics;
- catalog sections/public menu;
- vehicles and transit trips;
- reservations/check-in statistics and recent check-ins;
- marketplace aggregate statistics.

No backend route was deleted as a consequence of the frontend dead-code audit.

## Marketplace Prisma model scope

Models directly implicated by the active marketplace/business graph were traced through the schema relationships and route/service usage, including `BusinessProfile`, `BusinessCategoryImage`, `BusinessProduct`, `BusinessSubcategory`, `BusinessOrder`, `SmartEscrow`, `EscrowDispute`, `BusinessVerificationDocument`, `BusinessInvoice`, `BusinessReview`, `BusinessNotification`, `Reservation`, `TransitBooking`, `TransitTrip`, `DineInTab`, `DineInTabItem`, `BusinessFollower`, and supporting business/location/catalog entities.

All of these remain protected from deletion because there is active producer/consumer or relationship evidence. No Prisma model is classified as DEAD in this pass. Schema-wide model deletion requires a separate migration-aware reachability audit and is intentionally outside the frontend cleanup PR.

## Dynamic-navigation checks

For the two deleted files, the audit explicitly checked:

- concrete widget constructors;
- `Navigator.push` / `Navigator.maybePop` patterns;
- `showModalBottomSheet` usage;
- GoRouter/path-token references;
- notification/deep-link routing patterns;
- string/path searches for the screen/file identifiers;
- parent-child imports.

No hidden construction path was found for either deleted screen.

## Final result

The marketplace surface is not being treated as dead because it is large. The audit separates direct routing, parent-driven reachability, and proven orphans. Only two files met the proof threshold for deletion, and they were removed in a separate implementation PR after the audit documentation was established.
