# 03 — Marketplace Verticals: Shared Grammar, Retail, Restaurants, Hotels, Transit, Service Seam

> **Reconciled against live code — frontend `main` `6810007` (after #135 storefront shell, #136 Add Cash/typography), backend `main`, planning `main`.**
> Every identifier below is tagged **EXISTING** (`path:line` on live main) or **NEW** (introduced by this document). Anchors were re-verified with `git grep` on the live tree; this doc supersedes the `30188ef` snapshot version in PR #60. Review points from the PR #60 review that this document resolves:

| Review point | Resolution in this document |
|---|---|
| Anchors: v3 Phase D / `StoreSheetGeometry` / `collapsible_business_bar` | re-anchored on EXISTING `MarketStorefrontShell` + `MarketStorefrontSnaps`; bookmark/follow go through shell inputs/callbacks |
| `storeNavContextProvider` | replaced by `marketplaceSearchProvider.scope` (02 §9) |
| Raw hours parsing | service availability windows reuse `BusinessHours.stateOfRange` (02 §3) |
| Checkout/financial semantics | unchanged: every CTA routes to EXISTING order/booking flows (`onOrderProduct`, `hotelBooking`, `businessTransitTrips`); nothing here touches payment |

Implements brief §6 (vertical-by-vertical), §7.2 (identity morph), §5.6 (storefront continuity). Depends on the EXISTING store page `MarketStorefrontShell` (`lib/screens/marketplace/market_storefront_shell.dart:68`, landed in #135; snaps `MarketStorefrontSnaps {info .46, overview .62, shopping .78}`), hosted by `business_profile_screen.dart:1172`. This document only adds **content delivered through the shell's `productsBuilder` and callbacks** plus shared models; `MarketStorefrontSnaps` and the sheet's drag ownership are untouched.

Primary files:

- `lib/widgets/marketplace/marketplace_vertical_experience_stage.dart` (consumes new shared models; API additive)
- new `lib/marketplace/menu/menu_document.dart` (shared canonical menu model for flip-book + list)
- `lib/widgets/marketplace/restaurant_native_menu_journey.dart` + `restaurant_menu_journey_adapter.dart` (adapter consumes `MenuDocument`)
- `lib/marketplace/experiences/retail/*` (quick peek, sticky tray, recently viewed)
- `lib/widgets/marketplace/stay_summary_bar.dart`, `stay_date_ribbon.dart`, `room_dossier_content.dart` (stay decision UX)
- `lib/widgets/marketplace/transit_route_ribbon.dart`, `transit_seat_preview.dart`, `transit_boarding_pass.dart` + new `journey_thread.dart`
- new `lib/experience/gateways/service_flow_gateway.dart`

Financial/booking/checkout contracts (`storefront_retail_checkout_gateway.dart`, `retail_checkout.dart`, `hotel_booking.dart`, `transit_boarding.dart`, idempotency, holds) are **not** modified.

---

## 1. Shared storefront grammar (all worlds)

### 1.1 Store header contract inside the corrected sheet

The v3 §8 sheet gives: Banner → InfoStrip (behind) → Draggable StoreSheet → TopPill. Inside the sheet, the first content block is the same for every vertical:

```
StoreIdentityRow     logo (Hero AzIdentityTag.business) · name · TrustMark · category chip
StoreActionRow       primary intent (from MarketplaceExperienceCatalog.primaryActionLabel) · Save · Share · Follow
StoreContextRow      vertical-specific: order-mode switch / dates ribbon / route picker / collection chips
<vertical body>
```

New `lib/widgets/marketplace/store/store_identity_row.dart`:

```dart
class StoreIdentityRow extends ConsumerWidget {
  final BusinessProfile business;
  const StoreIdentityRow({super.key, required this.business});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final travel = AzMotion.of(context).travel;
    final cat = BusinessCategories.values.firstWhere((c) => c.wire == business.category, orElse: () => BusinessCategories.all);
    return Padding(
      padding: const EdgeInsets.fromLTRB(AzSpace.xl, AzSpace.md, AzSpace.xl, AzSpace.sm),
      child: Row(children: [
        AzIdentityMorph(
          tag: AzIdentityTag.business(business.bizId), travel: travel,
          child: ChatAvatar(imageUrl: business.logoUrl, name: business.businessName, size: 56),
        ),
        const SizedBox(width: AzSpace.md),
        Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
          Row(children: [
            Flexible(child: Text(business.businessName, style: AzText.titleL.copyWith(color: colors.textPrimary), maxLines: 2)),
            const SizedBox(width: AzSpace.xs),
            TrustMark(business: business, compact: true),
          ]),
          if (cat.wire.isNotEmpty) Text(cat.label + (business.subcategory != null ? ' · ${business.subcategory}' : ''),
              style: AzText.bodyS.copyWith(color: colors.textSecondary)),
        ])),
      ]),
    );
  }
}
```

The `logoUrl` here and the top pill's image are the **same** `business.logoUrl` (v3 §8.5). Never `coverImageUrl`, never `userProfilePictureUrl`.

### 1.2 `StoreActionRow`

```dart
class StoreActionRow extends ConsumerWidget {
  final BusinessProfile business;
  final VoidCallback onPrimary;
  const StoreActionRow({super.key, required this.business, required this.onPrimary});
  // Row: filled pill button (primaryActionLabel) · bookmark toggle bound to EXISTING savedBusinessesProvider (NEW widget StoreBookmarkToggle — collapsible_business_bar.dart is gone, nothing to reuse) · share → shell `onShare` · follow → shell `onToggleFollow` (`isFollowing` is a shell input; do not add a second follow source of truth)
}
```

Rule (v3 §10/brief §5.5): no "Storefront" button inside the canonical storefront. If `MarketplaceVerticalExperienceStage` currently renders one via `onOpenCatalogView`, that affordance moves into the catalog section header as "View all" only.

### 1.3 Store-scoped search filtering

`marketplaceSearchProvider` (02 §4) in `store` scope supplies `text`. Each vertical body applies the same predicate to its loaded data:

```dart
bool matchesStoreQuery(String q, {required String name, String? description, List<String> tags = const []}) {
  if (q.trim().isEmpty) return true;
  final l = q.toLowerCase();
  return name.toLowerCase().contains(l) || (description?.toLowerCase().contains(l) ?? false) || tags.any((t) => t.toLowerCase().contains(l));
}
```

Place in `lib/marketplace/store_query.dart`. Restaurants filter `MenuDocument.items`, retail filters products, hotels filter rooms, transit filters trips by destination.

### 1.4 Continuity (brief §5.6)

- Identity: `AzIdentityTag.business(bizId)` from `MerchantCard`/`CollapsibleBusinessBar` avatar → `StoreIdentityRow` logo → nav pill logo (v3 pill: wrap the pill avatar in `AzIdentityMorph` too — one tag, three sizes).
- Category context: `marketplaceSearchProvider.scope == MarketplaceSearchScope.store` (set by `business_profile_screen.dart`, 02 §9) — `storeNavContextProvider` does not exist.
- Saved/followed state: `savedBusinessesProvider` and the follow state already in `_FollowButton` — surfaced in `StoreActionRow`, no duplication.
- Cart state: `cartProvider` when `businessProfileId == business.id` → sticky tray (§2.4).

---

## 2. RETAIL

Existing: `lib/marketplace/experiences/retail/retail_experience.dart` (803), `retail_cart.dart`, `retail_cart_sheet.dart`, `retail_checkout.dart`, `retail_variant_swatches.dart`, `lib/widgets/marketplace/retail_dossier_picker.dart`, `retail_tray_commit.dart`, `lib/storefront/widgets/retail_collection_box_widget.dart`. Checkout/invoice semantics untouched.

### 2.1 Editorial product card

Replace the database-like tile in `retail_experience.dart`'s grid item builder (locate `GridView`/`SliverGrid` itemBuilder in the shelf section; the test `test/retail_shelf_test.dart` names the shelf) with `RetailProductCard`:

```dart
class RetailProductCard extends StatelessWidget {
  final BusinessProduct product;
  final AzamanColors colors;
  final bool saved;
  final VoidCallback onPeek;     // quick peek
  final VoidCallback onOpen;     // full detail
  final VoidCallback onToggleSave;
  const RetailProductCard({...});

  @override
  Widget build(BuildContext context) {
    final price = AzMoney.formatUsdcAsLocal(product.priceUsdc); // use the existing money formatter in lib/utils/az_money.dart
    return RepaintBoundary(child: GestureDetector(
      onTap: onOpen,
      onLongPress: () { AzamanHaptics.selection(); onPeek(); },
      child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
        AspectRatio(aspectRatio: 4 / 5, child: ClipRRect(
          borderRadius: BorderRadius.circular(AzRadius.lg),
          child: Stack(fit: StackFit.expand, children: [
            product.imageUrls.isNotEmpty
                ? AzamanNetworkImage(imageUrl: product.imageUrls.first, fit: BoxFit.cover)
                : Container(color: colors.softSurface),
            Positioned(top: AzSpace.sm, right: AzSpace.sm, child: _SaveDot(saved: saved, onTap: onToggleSave)),
            if (!product.isActive) Positioned.fill(child: Container(color: colors.background.withValues(alpha: .55),
                alignment: Alignment.center, child: Text('Unavailable', style: AzText.label))),
          ]),
        )),
        const SizedBox(height: AzSpace.sm),
        Text(product.name, style: AzText.body.copyWith(color: colors.textPrimary, fontWeight: FontWeight.w600), maxLines: 2),
        Text(price, style: AzText.money(14, color: colors.textPrimary)), // money stays Inter/tabular
      ]),
    ));
  }
}
```

No star counts, no "sold 120" (`totalOrders` is business analytics, not shopper truth). Stock appears only when a product exposes an authoritative stock field (none in `BusinessProduct` today → omit).

### 2.2 Quick peek (brief: "quick peek before full navigation")

Long-press or peek icon opens `AzamanSheet.showPanel` at `AzSheetGeometry.panelRestFraction` with `RetailQuickPeek`:

```
image carousel (PageView, dots)         ← reuse the dish image carousel pattern from restaurant_native_menu_journey.dart:339
name · price (Inter)
variant swatches (RetailVariantSwatches, existing)
quantity stepper
[ Add to cart ]   [ Full details → ]
```

`Add to cart` calls the **existing** add path (`onAddToTray(product, selections, quantity)` through `MarketplaceVerticalExperienceStage`) — the peek never computes price or stock itself. The sheet's identity: `AzIdentityMorph(AzIdentityTag... )` is *not* used (products are not identities; the Nest motion is the sheet itself).

### 2.3 Visual category chapters

`CatalogSection` already carries `name`, `imageUrl`, `displayOrder`. Render chapter headers as full-width 72dp bands (image if present, else accent fill) with the section name in `AzText.titleL`; a horizontal chapter chip rail pinned under `StoreActionRow` scrolls the sheet's controller to the chapter (`Scrollable.ensureVisible` on the chapter's `GlobalKey`, duration `MotionTokens.standard`, reduced motion → `jumpTo`). Owner: the sheet's `ScrollController` from `DraggableScrollableSheet` (v3 §8.2). No second controller.

### 2.4 Sticky purchase tray

Reuse `retail_tray_commit.dart`/`RetailCartSheet`. Visibility rule driven by `cartProvider`:

```dart
final showTray = cart.items.isNotEmpty && cart.businessProfileId == business.id;
```

Rendered as an `actionDock` surface docked above `AzSpace.navClearance`, inside the store `Stack` (outside the sheet) so it does not scroll. Appears/disappears with `AnimatedSlide` (`MotionTokens.control`, reduced motion → immediate). Tap → existing cart sheet.

### 2.5 Recently viewed (continuity)

In-memory per session: `lib/providers/recently_viewed_products_provider.dart` (`StateNotifier<List<String productId>>`, cap 12, no persistence). Rendered as a small rail at the end of the retail body only when non-empty and the ids resolve in the loaded catalog.

### 2.6 Lightweight comparison

Only when ≥ 2 products in the same `CatalogSection` share the same `tags` set and both have prices: a "Compare" chip in the chapter header opens a two-column sheet (name/price/variants/delivery terms from `BusinessProduct.deliveryTerms`/`estimatedDelivery`). Pure presentation of fields that exist; skipped otherwise.

---

## 3. RESTAURANTS

Existing: `RestaurantNativeMenuJourney` (flip-book), `restaurant_menu_journey_adapter.dart`, `restaurant_experience.dart`, `RestaurantOrderMode` + `restaurant_order_mode_switch.dart`, `restaurant_tray_rail.dart`, `restaurant_commit_surface.dart`, dine-in screens, reservations (`Reservation` model, `business_book_tab.dart`). Keep all behavior.

### 3.1 Canonical menu model — `lib/marketplace/menu/menu_document.dart`

Brief: "Make the flip-book and normal menu/list views share one canonical menu data model." Today the stage passes `menuSections: List<CatalogSection>`, `uncategorisedProducts`, `restaurantDishesById: Map<String, RestaurantDish>`, and the adapter re-derives pages. Introduce one immutable document both views consume.

```dart
import 'package:azaman/models/business_models.dart';

class MenuItemView {
  final BusinessProduct product;
  final RestaurantDish? dish;          // modifiers/options source when present
  final bool availableNow;             // from section availableFrom/To + product.isActive
  const MenuItemView({required this.product, required this.dish, required this.availableNow});
  String get id => product.id;
  List<String> get tags => product.tags;
}

class MenuChapter {
  final String id;
  final String title;
  final String? imageUrl;
  final String? availabilityLabel;     // "06:00 – 11:00" or null
  final List<MenuItemView> items;
  const MenuChapter({required this.id, required this.title, required this.imageUrl, required this.availabilityLabel, required this.items});
}

class MenuDocument {
  final List<MenuChapter> chapters;    // ordered by displayOrder; uncategorised last as "More"
  const MenuDocument(this.chapters);

  Iterable<MenuItemView> get items => chapters.expand((c) => c.items);
  MenuItemView? byId(String id) => items.cast<MenuItemView?>().firstWhere((i) => i!.id == id, orElse: () => null);

  MenuDocument filtered(String query) {
    if (query.trim().isEmpty) return this;
    final out = <MenuChapter>[];
    for (final c in chapters) {
      final items = c.items.where((i) => matchesStoreQuery(query, name: i.product.name, description: i.product.description, tags: i.tags)).toList();
      if (items.isNotEmpty) out.add(MenuChapter(id: c.id, title: c.title, imageUrl: c.imageUrl, availabilityLabel: c.availabilityLabel, items: items));
    }
    return MenuDocument(out);
  }

  static MenuDocument build({
    required List<CatalogSection> sections,
    required List<BusinessProduct> uncategorised,
    required Map<String, RestaurantDish> dishesById,
    required DateTime now,
  }) {
    final sorted = [...sections]..sort((a, b) => a.displayOrder.compareTo(b.displayOrder));
    MenuItemView view(BusinessProduct p, bool sectionOpen) =>
        MenuItemView(product: p, dish: dishesById[p.id], availableNow: p.isActive && sectionOpen);
    bool sectionOpen(CatalogSection s) {
      if (s.availableFrom == null || s.availableTo == null) return true;
      return BusinessHours.stateOfRange('${s.availableFrom}-${s.availableTo}', now) != OpenState.closed; // 02 §3 helper; same parser as store hours
    }
    final chapters = <MenuChapter>[
      for (final s in sorted.where((s) => s.isActive))
        MenuChapter(
          id: s.id, title: s.name, imageUrl: s.imageUrl,
          availabilityLabel: (s.availableFrom != null && s.availableTo != null) ? '${s.availableFrom} – ${s.availableTo}' : null,
          items: s.products.map((p) => view(p, sectionOpen(s))).toList(),
        ),
      if (uncategorised.isNotEmpty)
        MenuChapter(id: '__more', title: 'More', imageUrl: null, availabilityLabel: null, items: uncategorised.map((p) => view(p, true)).toList()),
    ];
    return MenuDocument(chapters);
  }
}
```

Provider: `menuDocumentProvider(bizId)` built from the data the stage already receives (or computed once in the store screen and passed down). **Build once per data change, not per frame** — memoize on the identity of `sections`/`products` lists.

### 3.2 Adapter change

`restaurant_menu_journey_adapter.dart` currently maps `CatalogSection` → `_MenuPage`. Change its input to `MenuDocument` and map `MenuChapter → page`, `MenuItemView → dish card`. `RestaurantNativeMenuJourney`'s constructor adds an alternate factory:

```dart
factory RestaurantNativeMenuJourney.fromDocument({
  required String businessName,
  required MenuDocument document,
  required AzamanColors colors,
  required void Function(BusinessProduct, Map<String, String>, int) onAddToTray,
}) => RestaurantNativeMenuJourney(
      businessName: businessName,
      sections: const [], uncategorisedProducts: const [], dishesById: const {}, // legacy params kept
      document: document, colors: colors, onAddToTray: onAddToTray,
    );
```

and internally prefers `document` when non-null. Existing callers keep compiling; `MarketplaceVerticalExperienceStage` switches to `fromDocument`. The list view (`restaurant_experience.dart`) switches to iterate `document.chapters`. Existing tests under `test/marketplace/experiences/restaurant/` must keep passing — add `test/marketplace/menu/menu_document_test.dart` (ordering, availability window, uncategorised → More, filter).

### 3.3 Menu view toggle

Under `StoreActionRow`, a two-state segmented toggle **Flip-book | List** (persisted per user in `SharedPreferences` key `az_menu_view_mode`). Both read the same `MenuDocument`; switching is a cross-fade `MotionTokens.standard` (reduced motion → cut). The flip-book's own page gestures are horizontal; the sheet's drag is vertical — no conflict.

### 3.4 Order mode + dine-in

`RestaurantOrderMode` switch stays in `StoreContextRow`. Show only the modes the business supports: `MarketplaceExperienceCatalog.restaurant.capabilities` ∩ business capability flags (check `experience` map passed to the stage for `pickup/delivery/dineIn` keys; if absent, show the switch exactly as today — do not infer).

### 3.5 Meal path (optional, deterministic)

If `MenuDocument.chapters` contains chapters whose titles match `/starter|main|side|drink|dessert/i`, show a quiet "Build a meal" chip rail that scrolls to each chapter in order. It is a navigation aid over existing chapters; no new animation beyond `Scrollable.ensureVisible`.

---

## 4. HOTELS

Existing: `hotel_booking_screen.dart`, `lib/marketplace/experiences/hotel/hotel_experience.dart`, `hotel_booking.dart`, `StaySummaryBar{room, checkIn, checkOut, nights}`, `stay_date_ribbon.dart`, `room_dossier_content.dart`, `hotel_floor_plan_preview.dart`, `building_cross_section.dart`, `hotel_arrival_sheet.dart`, `booking_success_sheet.dart`, `lib/models/hotel_models.dart`, `hotel_marketplace_provider.dart`. Booking semantics untouched.

### 4.1 Stay decision state (UI-only)

New `lib/marketplace/experiences/hotel/stay_decision.dart`:

```dart
class StayDecision {
  final DateTime? checkIn;
  final DateTime? checkOut;
  final int guests;
  final String? roomId;
  const StayDecision({this.checkIn, this.checkOut, this.guests = 1, this.roomId});
  int? get nights => (checkIn != null && checkOut != null) ? checkOut!.difference(checkIn!).inDays : null;
  bool get datesChosen => nights != null && nights! > 0;
  bool get complete => datesChosen && roomId != null;
  StayDecision copyWith({...}) => ...;
}
final stayDecisionProvider = StateProvider.autoDispose.family<StayDecision, String>((ref, bizId) => const StayDecision());
```

The **authoritative** price/availability still comes from the hotel booking provider; `StayDecision` only records what the user picked and drives what is shown.

### 4.2 Store context row for hotels

`StayDateRibbon` (existing) sits in `StoreContextRow`; tapping opens the existing date picker. Guests stepper beside it. When `datesChosen` becomes true, the room list re-requests availability through the **existing** provider call (not from the widget — from a `ref.listen` in the stage's notifier).

### 4.3 Room cards for effortless comparison

Fixed column grammar per card so the eye can scan vertically: image (16:10) · room name (`AzText.title`) · 2 amenity chips max · availability line (only from provider) · price per night (`AzText.money`, Inter) · "Select". Identical height via `IntrinsicHeight` in a 1-column list. Unavailable rooms stay visible with "Not available for these dates" (truthful), not hidden.

### 4.4 Persistent booking summary

`StaySummaryBar` becomes the hotel's `actionDock` (same slot as the retail tray, §2.4): visible when `stayDecision.complete`, docked above nav clearance, never overlapping content (the sheet's bottom padding grows by the bar height via `AzSpace.navClearanceHeight + 72`).

### 4.5 Policy presentation

Cancellation policy, check-in/out times, deposit terms: render as a plain `InfoStrip`-style list (hairline separators, no cards) **inside** the room dossier before "Book". Never collapsed by default. Text comes from `hotel_models.dart` fields; if a field is null, the row is omitted — never a generic placeholder sentence.

### 4.6 Stay story (visual sequencing)

Dossier image carousel + a tiny 4-step indicator (Dates → Room → Review → Confirmed) at the top of the dossier, derived from `StayDecision` + booking state. Reduced motion → static indicator. No new timers.

---

## 5. TRANSIT

Existing: `transit_trip_list_screen.dart`, `transit_seat_selection_screen.dart`, `TransitRouteRibbon{trips, colors, onTripTap}`, `transit_seat_preview.dart`, `transit_hold_ring` (tests), `TransitBoardingPassCard{pass, colors}`, `lib/marketplace/experiences/transit/*` (hold gateway, boarding), `marketplace_booking_models.dart`. Hold/booking/idempotency untouched.

### 5.1 Journey thread (brief §6.4) — `lib/widgets/marketplace/transit/journey_thread.dart`

A thin vertical route line that persists across trip list → seat selection → booking → boarding pass, changing *state* only (visual continuity, not a booking source).

```dart
enum JourneyStage { search, trip, seat, booking, boarded }

class JourneyThreadModel {
  final String? fromLabel;
  final String? toLabel;
  final DateTime? departAt;
  final String? seatLabel;
  final JourneyStage stage;
  const JourneyThreadModel({this.fromLabel, this.toLabel, this.departAt, this.seatLabel, required this.stage});
}

/// UI-only. Derived in each screen from the authoritative objects it already has
/// (trip, hold, pass). Never persisted; never read back to make decisions.
final journeyThreadProvider = StateProvider.autoDispose<JourneyThreadModel?>((_) => null);

class JourneyThread extends StatelessWidget {
  final JourneyThreadModel model;
  final AzamanColors colors;
  const JourneyThread({super.key, required this.model, required this.colors});

  @override
  Widget build(BuildContext context) {
    final steps = JourneyStage.values;
    final idx = steps.indexOf(model.stage);
    return RepaintBoundary(child: SizedBox(
      height: 44,
      child: CustomPaint(
        painter: _ThreadPainter(progress: idx / (steps.length - 1), color: colors.accent, track: colors.softSurface),
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: AzSpace.xl),
          child: Row(children: [
            Expanded(child: Text(model.fromLabel ?? '—', style: AzText.label.copyWith(color: colors.textPrimary), maxLines: 1, overflow: TextOverflow.ellipsis)),
            Icon(Icons.arrow_forward_rounded, size: 14, color: colors.textTertiary),
            Expanded(child: Text(model.toLabel ?? '—', textAlign: TextAlign.end, style: AzText.label.copyWith(color: colors.textPrimary), maxLines: 1, overflow: TextOverflow.ellipsis)),
            if (model.seatLabel != null) ...[const SizedBox(width: AzSpace.sm), Text('Seat ${model.seatLabel}', style: AzText.caption.copyWith(color: colors.textSecondary))],
          ]),
        ),
      ),
    ));
  }
}

class _ThreadPainter extends CustomPainter {
  final double progress; final Color color; final Color track;
  const _ThreadPainter({required this.progress, required this.color, required this.track});
  @override
  void paint(Canvas c, Size s) {
    final y = s.height - 4;
    final p = Paint()..strokeWidth = 2..strokeCap = StrokeCap.round;
    c.drawLine(Offset(AzSpace.xl, y), Offset(s.width - AzSpace.xl, y), p..color = track);
    c.drawLine(Offset(AzSpace.xl, y), Offset(AzSpace.xl + (s.width - 2 * AzSpace.xl) * progress, y), p..color = color);
  }
  @override
  bool shouldRepaint(_ThreadPainter o) => o.progress != progress || o.color != color || o.track != track;
}
```

Placement: pinned under the store identity in the transit store; at the top of seat selection and booking screens; above the boarding pass card. Progress change animates with `TweenAnimationBuilder` (`MotionTokens.standard`; reduced motion → `Duration.zero`).

### 5.2 Hierarchy: where · when · seat · price · status

In `transit_trip_list_screen.dart` each trip node (`_RibbonNode`/`_ExpandedTripCard` in `transit_route_ribbon.dart`) is reordered to: destination (title) → departure time (`AzText.titleL`, Inter) → operator/bus → price (`AzText.money`) → seats-left line **only if** the trip model exposes an authoritative count. Status chips (On time/Boarding/Departed) only from server fields. No "popular route" badges.

### 5.3 Seat selection

Keep `transit_seat_preview.dart` and the hold ring. Add the `JourneyThread` header and a bottom `actionDock` showing "Seat 12A · GH₵ 45.00 · Hold 04:12" from the existing hold state. The hold countdown is a **semantic** timer (it reflects server expiry) and is allowed; it reads the server `expiresAt` and ticks once per second via a single `Ticker`, disposed with the screen.

### 5.4 Boarding pass

`TransitBoardingPassCard` unchanged visually. Above it the thread at `JourneyStage.boarded`; after a successful booking the "Seal" motion is a single 350 ms scale-in of the pass + `AzamanHaptics.moneyLanded()` (existing). No loop.

---

## 6. LOGISTICS / SERVICE SEAM — `lib/experience/gateways/service_flow_gateway.dart`

Brief §6.5: do not fake. Provide the typed contract so future service verticals compose with the same store grammar.

```dart
enum ServiceFlowStage { request, slot, location, estimate, confirm, live, complete }

class ServiceSlot { final DateTime start; final DateTime end; final bool available; const ServiceSlot(...); }
class ServiceEstimate { final double amountUsdc; final String? note; const ServiceEstimate(...); }
class ServiceRequestDraft { final String bizId; final String? serviceId; final ServiceSlot? slot; final String? locationId; final String? notes; const ServiceRequestDraft(...); }

abstract interface class ServiceFlowGateway {
  Set<ServiceFlowStage> get supportedStages; // empty today
  Future<AzGatewayResult<List<ServiceSlot>>> slots(String bizId, DateTime day);
  Future<AzGatewayResult<ServiceEstimate>> estimate(ServiceRequestDraft draft);
  Future<AzGatewayResult<String>> confirm(ServiceRequestDraft draft, {required String idempotencyKey});
  Stream<ServiceFlowStage> liveStatus(String requestId);
}

/// Default binding: nothing supported. UI checks `supportedStages` and renders
/// the existing `service_experience_stage.dart` fallback.
final serviceFlowGatewayProvider = Provider<ServiceFlowGateway>((_) => const UnsupportedServiceFlowGateway());
```

`UnsupportedServiceFlowGateway` returns `AzUnsupported` for everything. `LOGISTICS` wire remains Transit; no category semantics change.

---

## 7. Store tests (M2)

- `menu_document_test.dart` (above).
- `test/marketplace/experiences/retail/retail_quick_peek_test.dart`: long-press opens panel; Add calls `onAddToTray` with selections/quantity; no price computed in the peek.
- `test/marketplace/experiences/retail/retail_tray_visibility_test.dart`: tray only when cart belongs to this business.
- `test/marketplace/experiences/hotel/stay_decision_test.dart`: nights/complete derivations; summary bar visibility.
- `test/widgets/marketplace/journey_thread_test.dart`: progress per stage; reduced motion renders final state after one pump.
- Store search filter test: `matchesStoreQuery` + `MenuDocument.filtered`.
- Goldens (brief §26 #4–7) through `pumpGoldenSurface` with fixture business per vertical; the sheet geometry goldens belong to `market_storefront_shell` (#135) and are not duplicated here.