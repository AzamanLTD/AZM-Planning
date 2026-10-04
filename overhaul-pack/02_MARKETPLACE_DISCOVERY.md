# 02 — Marketplace Discovery: Portal, Unified Search, Merchant Cards, Signals

> **Reconciled against live code — frontend `main` `6810007` (after #135 storefront shell, #136 Add Cash/typography), backend `main`, planning `main`.**
> Every identifier below is tagged **EXISTING** (`path:line` on live main) or **NEW** (introduced by this document). Anchors were re-verified with `git grep` on the live tree; this doc supersedes the `30188ef` snapshot version in PR #60. Review points from the PR #60 review that this document resolves:

| Review point | Resolution in this document |
|---|---|
| `BusinessHours` not on main; `businessMeta['hours']` raw-map leak | §3 — NEW `lib/utils/business_hours.dart` built from EXISTING `BusinessLocation.operatingHours` + `business_card.dart:374`; widgets call `business.openStateAt(now)` only |
| `storeNavContextProvider` / `PortalContext` / `MarketplaceSearchScope` not on main | `MarketplaceSearchScope` is NEW (introduced §4); nav wiring replaced by `MarketplaceSearchBinding` seam (§4.1) and the store hand-off re-anchored on `business_profile_screen.dart:1172` (§9) |
| `AzRoutes` vs `AzRouteNames` | both EXIST; paths come from `AzRoutes` (`businessProfile(bizId)`, `savedBusinesses`); `AzRoutes.cart` is NEW and must be registered |
| `AzRadius.card` | → EXISTING `AzRadius.lg` |

Implements brief §5 (marketplace redesign), §7 (unique features), §11 (one system). Depends on 01 (`AzIntent`, `AzIdentityTag`, gateways). It does **not** depend on nav work: the search state in §4 is self-owned and exposes a `MarketplaceSearchBinding` seam for the future in-pill search field (not on live main).

Primary files touched:

- `lib/screens/marketplace/marketplace_home_screen.dart` (portal body rebuilt; explore machinery kept)
- `lib/providers/business_provider.dart` (no changes to `BusinessSearchNotifier`; new providers in new files)
- new `lib/providers/marketplace_search_provider.dart`
- new `lib/providers/marketplace_world_memory_provider.dart`
- new `lib/providers/marketplace_resume_provider.dart`
- new `lib/experience/gateways/marketplace_discovery_gateway.dart`
- new `lib/widgets/marketplace/discovery/*.dart` (intent rail, world deck, utility rail, local pulse, resume card, merchant card, trust mark)

Nothing in `lib/storefront/**`, `BusinessService` or invoice/order code changes.

---

## 1. Target information architecture

```
MarketplaceHomeScreen (mode = portal)              ← CustomScrollView, one owner
├── SliverToBoxAdapter  DiscoveryHeader             "Discover" / "Around you" + inline search hint (opens nav search)
├── SliverToBoxAdapter  ResumeCard?                 "Pick up where you left off" — only when real state exists
├── SliverToBoxAdapter  IntentRail                  Eat · Shop · Ride · Stay · Nearby · Saves · Recent
├── SliverToBoxAdapter  WorldDeck                   4 world cards (Transit/Restaurants/Hotels/Retail) with real signals
├── SliverToBoxAdapter  UtilityRail                 Near me · Open now · Top rated · Saved · New (Deals only if offer model exists)
├── SliverToBoxAdapter  MerchantStoriesRow          reuses MarketplaceExpandedStories
├── SliverToBoxAdapter  LocalPulse?                 open now / nearby / departing soon — nullable inputs
├── SliverToBoxAdapter  FeaturedRail                existing _featuredRail
└── SliverToBoxAdapter  ExploreAllButton            existing _portalExploreAll
```

Explore mode is unchanged except: its header search field is removed in favor of the nav pill search (v3 B) and both read `marketplaceSearchProvider`.

---

## 2. Discovery gateway — `lib/experience/gateways/marketplace_discovery_gateway.dart`

Wraps the existing providers behind one typed surface so the portal never calls `BusinessService` directly and so demo/preview adapters are possible.

```dart
import 'package:azaman/models/business_models.dart';
import 'az_gateway_result.dart';

enum DiscoverySignal { openNow, nearby, topRated, recentlyAdded, departingSoon, availableTonight }

/// Immutable snapshot the portal renders from. Every list may be empty; the
/// UI must render nothing for empty lists (no fake richness).
class DiscoverySnapshot {
  final List<BusinessProfile> catalog;      // businessSearchProvider.results (whole catalog in portal)
  final List<BusinessProfile> featured;     // featuredBusinessesProvider
  final Map<String, double>? distanceKmByBizId; // nearbySearchProvider, null when no location
  final Set<String> savedBizIds;            // savedBusinessesProvider
  final DateTime now;
  const DiscoverySnapshot({
    required this.catalog,
    required this.featured,
    required this.distanceKmByBizId,
    required this.savedBizIds,
    required this.now,
  });
}

abstract interface class MarketplaceDiscoveryGateway {
  /// Which signals this build can compute truthfully.
  Set<DiscoverySignal> get supportedSignals;

  Future<AzGatewayResult<void>> refreshCatalog();
  Future<AzGatewayResult<void>> refreshNearby();
}
```

Real adapter (`lib/experience/gateways/http_marketplace_discovery_gateway.dart`):

```dart
class RiverpodMarketplaceDiscoveryGateway implements MarketplaceDiscoveryGateway {
  final Ref ref;
  RiverpodMarketplaceDiscoveryGateway(this.ref);

  @override
  Set<DiscoverySignal> get supportedSignals => const {
        DiscoverySignal.openNow,      // derived from BusinessLocation.operatingHours via BusinessHours (§3) — never from businessMeta
        DiscoverySignal.nearby,       // nearbySearchProvider distances
        DiscoverySignal.topRated,     // averageRating + reviewCount >= 5
        DiscoverySignal.recentlyAdded // only if BusinessProfile gains createdAt; else drop from set
      };

  @override
  Future<AzGatewayResult<void>> refreshCatalog() async {
    try {
      await ref.read(businessSearchProvider.notifier).search(query: '', category: null);
      return const AzOk(null);
    } catch (e) {
      return AzFailed('Could not refresh businesses', cause: e);
    }
  }

  @override
  Future<AzGatewayResult<void>> refreshNearby() async {
    // Mirror MarketplaceHomeScreen._fireNearby: location permission → searchNearby.
    ...
  }
}

final marketplaceDiscoveryGatewayProvider = Provider<MarketplaceDiscoveryGateway>(
  (ref) => RiverpodMarketplaceDiscoveryGateway(ref),
);
```

> `recentlyAdded` and `departingSoon`/`availableTonight` require fields the snapshot does not expose (`BusinessProfile` has no `createdAt`; transit departures are per-business trip data). The adapter **omits** them from `supportedSignals`, and every widget below hides a signal that is not supported. Do not add a fake.

Snapshot provider (`lib/providers/marketplace_discovery_provider.dart`):

```dart
final discoverySnapshotProvider = Provider<DiscoverySnapshot>((ref) {
  final search = ref.watch(businessSearchProvider);
  final featured = ref.watch(featuredBusinessesProvider);
  final nearby = ref.watch(nearbySearchProvider);
  final saved = ref.watch(savedBusinessesProvider);
  return DiscoverySnapshot(
    catalog: search.results,
    featured: featured,
    distanceKmByBizId: nearby.results.isEmpty
        ? null
        : {for (final r in nearby.results) r.business.bizId: r.distanceKm}, // adapt to NearbySearchState shape
    savedBizIds: saved,
    now: DateTime.now(),
  );
});
```

(Check `NearbySearchState` field names in `business_provider.dart:223` before writing the map comprehension.)

---

## 3. "Open now" derivation (shared, deterministic)

**EXISTING:** the only hours parser on live main is `lib/widgets/business_card.dart:374 _isOpenNow()` / `_parseTime`, reading `BusinessLocation.operatingHours` (`lib/models/business_models.dart:636`, shape `{mon: "8:00-22:00", ...}`, nullable, keyed `sun..sat` via `DateTime.weekday % 7`). `collapsible_business_bar.dart` no longer exists and `businessMeta` never carried hours (it carries `showcaseUrls`). Extract the parser into one pure helper so cards, pulse, world deck and the store info strip agree, and delete `_isOpenNow` from `business_card.dart`.

**NEW** `lib/utils/business_hours.dart`:

```dart
import '../models/business_models.dart'; // EXISTING: BusinessProfile, BusinessLocation

enum OpenState { open, closed, unknown }

/// Pure, clock-injected. The ONLY place that understands `operatingHours`.
class BusinessHours {
  const BusinessHours._();

  // Same key mapping as the legacy `_isOpenNow` (Sunday = 0).
  static const _days = ['sun', 'mon', 'tue', 'wed', 'thu', 'fri', 'sat'];

  /// A business is open if ANY of its locations is open right now.
  /// `unknown` when no location carries parseable hours for today.
  static OpenState stateAt(List<BusinessLocation> locations, DateTime now) {
    var sawHours = false;
    for (final loc in locations) {
      final hours = loc.operatingHours;
      if (hours == null || hours.isEmpty) continue;
      final s = stateOfRange(hours[_days[now.weekday % 7]]?.toString(), now);
      if (s == OpenState.unknown) continue;
      sawHours = true;
      if (s == OpenState.open) return OpenState.open;
    }
    return sawHours ? OpenState.closed : OpenState.unknown;
  }

  /// Parses one `"HH:mm-HH:mm"` window (hour may be 1 or 2 digits, as the
  /// data has `8:00-22:00`). Overnight windows (`20:00-02:00`) are honoured.
  static OpenState stateOfRange(String? range, DateTime now) {
    if (range == null) return OpenState.unknown;
    final parts = range.split('-');
    if (parts.length != 2) return OpenState.unknown;
    final open = _minutes(parts[0].trim());
    final close = _minutes(parts[1].trim());
    if (open == null || close == null) return OpenState.unknown;
    final nowMins = now.hour * 60 + now.minute;
    final inWindow = close < open
        ? (nowMins >= open || nowMins < close)   // crosses midnight
        : (nowMins >= open && nowMins < close);
    return inWindow ? OpenState.open : OpenState.closed;
  }

  static int? _minutes(String t) {
    final m = RegExp(r'^(\d{1,2}):(\d{2})$').firstMatch(t);
    if (m == null) return null;
    final h = int.parse(m.group(1)!), mm = int.parse(m.group(2)!);
    if (h > 24 || mm > 59) return null;
    return h * 60 + mm;
  }
}

extension BusinessOpenState on BusinessProfile {
  OpenState openStateAt(DateTime now) => BusinessHours.stateAt(locations, now);
}
```

Then in `business_card.dart` replace the `_isOpenNow()` body with `business.openStateAt(DateTime.now()) == OpenState.open` and delete `_parseTime`. Add `test/utils/business_hours_test.dart` covering: normal window, overnight window, single-digit hour, missing day key, malformed string, `null` hours on one location but open on another, all-null → `unknown`. No widget in this pack reads `operatingHours` or `businessMeta` directly — they call `openStateAt`.

---

## 4. Unified search state — `lib/providers/marketplace_search_provider.dart`

Brief §5.3: the nav pill morph (v3 B) and the explore screen must drive **one** state. Today `MarketplaceHomeScreen` keeps `_searchExpanded` + a local `TextEditingController` and debounces into `businessSearchProvider`. Replace the local state with this notifier; the screen and the nav pill both watch it.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';

enum MarketplaceSearchScope { marketplace, world, store }

class MarketplaceSearchState {
  final String text;
  final bool focused;
  final MarketplaceSearchScope scope;
  final String? worldWire;        // when scope == world
  final String? storeBizId;       // when scope == store
  final List<String> recent;      // last 8, persisted
  final List<String> suggestions; // derived, never persisted
  const MarketplaceSearchState({
    this.text = '',
    this.focused = false,
    this.scope = MarketplaceSearchScope.marketplace,
    this.worldWire,
    this.storeBizId,
    this.recent = const [],
    this.suggestions = const [],
  });

  bool get isActive => focused || text.isNotEmpty;

  MarketplaceSearchState copyWith({...}) => ...; // standard
}

class MarketplaceSearchNotifier extends StateNotifier<MarketplaceSearchState> {
  MarketplaceSearchNotifier(this.ref) : super(const MarketplaceSearchState()) {
    _loadRecent();
  }
  final Ref ref;
  static const _kRecent = 'az_marketplace_recent_searches';

  Future<void> _loadRecent() async {
    final prefs = await SharedPreferences.getInstance();
    state = state.copyWith(recent: prefs.getStringList(_kRecent) ?? const []);
  }

  void setScope(MarketplaceSearchScope scope, {String? worldWire, String? storeBizId}) {
    state = state.copyWith(scope: scope, worldWire: worldWire, storeBizId: storeBizId, text: '', focused: false);
  }

  void focus(bool value) => state = state.copyWith(focused: value);

  void changed(String text) {
    state = state.copyWith(text: text, suggestions: _suggest(text));
  }

  /// Commit: persist recent, push into the existing search plumbing.
  Future<void> submit() async {
    final q = state.text.trim();
    if (q.isEmpty) return;
    final recent = [q, ...state.recent.where((r) => r != q)].take(8).toList();
    state = state.copyWith(recent: recent, focused: false);
    final prefs = await SharedPreferences.getInstance();
    await prefs.setStringList(_kRecent, recent);
    switch (state.scope) {
      case MarketplaceSearchScope.marketplace:
      case MarketplaceSearchScope.world:
        await ref.read(businessSearchProvider.notifier).search(query: q, category: state.worldWire);
      case MarketplaceSearchScope.store:
        // Store-scoped search filters the store's loaded catalog locally (03 §1.3); nothing to fetch.
        break;
    }
  }

  void clear() => state = state.copyWith(text: '', suggestions: const []);

  List<String> _suggest(String text) {
    if (text.length < 2) return state.recent.take(4).toList();
    final lower = text.toLowerCase();
    final cats = BusinessCategories.values.where((c) => c.label.toLowerCase().contains(lower)).map((c) => c.label);
    final names = ref.read(businessSearchProvider).results
        .where((b) => b.businessName.toLowerCase().contains(lower))
        .map((b) => b.businessName)
        .take(5);
    return {...cats, ...names}.toList();
  }
}

final marketplaceSearchProvider =
    StateNotifierProvider<MarketplaceSearchNotifier, MarketplaceSearchState>(
  (ref) => MarketplaceSearchNotifier(ref),
);
```

### 4.1 Surgical change in `marketplace_home_screen.dart`

Anchor (state fields, ~L106–L120):

```dart
  _MarketplaceHomeMode _mode = _MarketplaceHomeMode.portal;
  ...
  bool _searchExpanded = false;
```

Change:

1. Delete `_searchExpanded`, the local search `TextEditingController` and its debounce `Timer`. Replace every `_searchExpanded` read with `ref.watch(marketplaceSearchProvider.select((s) => s.isActive))` and every `_closeSearch()` with `ref.read(marketplaceSearchProvider.notifier).clear()`.
2. `_enterExplore(wire)` additionally calls `ref.read(marketplaceSearchProvider.notifier).setScope(MarketplaceSearchScope.world, worldWire: wire)`; `_returnToPortal()` calls `setScope(MarketplaceSearchScope.marketplace)`.
3. The slide-out header search widget in explore mode (anchor: `offset: _searchExpanded ? const Offset(-0.2, 0) : Offset.zero` ~L949) is removed. Explore keeps the filter/sort/map controls only.
4. **Binding seam (nav-independent).** There is no search field in `ContextualNavBand` / `PremiumBottomNav` on live main. The provider is therefore bound to the explore-mode `TextField` that exists today, and exposes a small **NEW** binding object so the future in-pill field plugs in without touching the provider:

```dart
/// lib/providers/marketplace_search_binding.dart (NEW)
class MarketplaceSearchBinding {
  MarketplaceSearchBinding(this.ref);
  final WidgetRef ref;
  MarketplaceSearchNotifier get _n => ref.read(marketplaceSearchProvider.notifier);
  void onChanged(String text) => _n.changed(text);
  void onSubmit(String _) => _n.submit();
  void onFocus(bool focused) => _n.focus(focused);
  List<String> placeholders(MarketplaceSearchState s, {String? storeName}) =>
      MarketplacePlaceholders.forScope(s.scope, world: s.worldWire, storeName: storeName);
}
```

Today the explore `TextField` passes `onChanged`/`onSubmitted`/`focusNode.addListener` straight into this binding. When the nav gains a search field (future slice), it instantiates the same binding — the provider, placeholders and back semantics do not change.

Back semantics (brief §17) now fall out: back (system back today; pill back when it exists) → if `search.isActive` → `clear()`; else if store route → pop; else `_returnToPortal()`. Implement this as a `PopScope` in `MarketplaceHomeScreen`, so the order of precedence is tested once.

### 4.2 Placeholders (data-driven, v3 §5.4)

`lib/widgets/marketplace/discovery/marketplace_placeholders.dart`:

```dart
abstract final class MarketplacePlaceholders {
  static const marketplace = ['Search restaurants near you', 'Find a store or vendor', 'Try "jollof" or "barber"', 'Search places on Azaman'];
  static List<String> forScope(MarketplaceSearchScope scope, {String? world, String? storeName}) {
    final s = storeName ?? 'this store';
    return switch ((scope, world)) {
      (MarketplaceSearchScope.store, 'FOOD_BEVERAGE') => ['Search for food in "$s"', 'Find a dish at $s', "Hungry? Search $s's menu"],
      (MarketplaceSearchScope.store, 'RETAIL') => ['Search products in "$s"', 'What are you looking for at $s?'],
      (MarketplaceSearchScope.store, 'HOSPITALITY') => ['Search rooms at "$s"', 'Dates, room type…'],
      (MarketplaceSearchScope.store, 'LOGISTICS') => ['Search trips from "$s"', 'Where are you going?'],
      (MarketplaceSearchScope.world, 'FOOD_BEVERAGE') => ['Search restaurants', 'Try "waakye" or "pizza"'],
      (MarketplaceSearchScope.world, 'RETAIL') => ['Search shops and products'],
      (MarketplaceSearchScope.world, 'HOSPITALITY') => ['Search hotels and stays'],
      (MarketplaceSearchScope.world, 'LOGISTICS') => ['Search routes and operators'],
      _ => marketplace,
    };
  }
}
```

---

## 5. World memory — `lib/providers/marketplace_world_memory_provider.dart`

Brief §7.1. In-memory (not persisted) per-world UI context: selected category, query, scroll offset. **Never** stores transactional state — carts live in `cartProvider`, resume reads them read-only (§6).

```dart
class WorldMemory {
  final String? query;
  final String? subcategory;
  final double scrollOffset;
  final DateTime touchedAt;
  const WorldMemory({this.query, this.subcategory, this.scrollOffset = 0, required this.touchedAt});
  bool isFresh(DateTime now) => now.difference(touchedAt) < const Duration(minutes: 30);
}

class WorldMemoryNotifier extends StateNotifier<Map<String, WorldMemory>> {
  WorldMemoryNotifier() : super(const {});
  void remember(String wire, {String? query, String? subcategory, double? scrollOffset}) {
    final prev = state[wire];
    state = {
      ...state,
      wire: WorldMemory(
        query: query ?? prev?.query,
        subcategory: subcategory ?? prev?.subcategory,
        scrollOffset: scrollOffset ?? prev?.scrollOffset ?? 0,
        touchedAt: DateTime.now(),
      ),
    };
  }
  WorldMemory? recall(String wire) {
    final m = state[wire];
    return (m != null && m.isFresh(DateTime.now())) ? m : null;
  }
}

final worldMemoryProvider = StateNotifierProvider<WorldMemoryNotifier, Map<String, WorldMemory>>((_) => WorldMemoryNotifier());
```

In `MarketplaceHomeScreen`:
- `_enterExplore(wire)`: after `setScope`, `final m = ref.read(worldMemoryProvider.notifier).recall(wire); if (m?.query != null) { changed(m.query); submit(); }`. Scroll restore only when the result list is non-empty and the offset ≤ `maxScrollExtent` (restore in a post-frame callback, `jumpTo`, no animation).
- `_returnToPortal()`: `remember(wire, query: search.text, scrollOffset: _listController.offset)`.

Test: remember/recall, freshness expiry, recall of unknown wire → null.

---

## 6. Resume — `lib/providers/marketplace_resume_provider.dart`

Brief §7.4. Derived, never stored. Shows at most one card.

```dart
enum ResumeKind { cart, worldSearch, savedHotel }

class ResumeIntent {
  final ResumeKind kind;
  final String title;      // "Finish your order at Kofi's Kitchen"
  final String subtitle;   // "3 items · GH₵ 86.00"
  final String? bizId;
  final String? worldWire;
  const ResumeIntent({required this.kind, required this.title, required this.subtitle, this.bizId, this.worldWire});
}

final marketplaceResumeProvider = Provider<ResumeIntent?>((ref) {
  final cart = ref.watch(cartProvider);
  if (cart.items.isNotEmpty && !cart.isCheckingOut) {
    final count = cart.items.fold<int>(0, (a, i) => a + i.quantity);
    return ResumeIntent(
      kind: ResumeKind.cart,
      title: 'Finish your order at ${cart.businessName}',
      subtitle: '$count item${count == 1 ? '' : 's'}',   // amount formatting via existing AzMoney helpers if CartState exposes a total
      bizId: cart.businessProfileId,
    );
  }
  final memories = ref.watch(worldMemoryProvider);
  final fresh = memories.entries.where((e) => e.value.isFresh(DateTime.now()) && (e.value.query?.isNotEmpty ?? false)).toList()
    ..sort((a, b) => b.value.touchedAt.compareTo(a.value.touchedAt));
  if (fresh.isNotEmpty) {
    final e = fresh.first;
    final label = BusinessCategories.values.firstWhere((c) => c.wire == e.key).label;
    return ResumeIntent(kind: ResumeKind.worldSearch, title: 'Back to "${e.value.query}"', subtitle: 'in $label', worldWire: e.key);
  }
  return null;
});
```

(Confirm `CartState` field names — `businessName`, `businessProfileId`, `isCheckingOut` exist per `cart_provider.dart:96`.)

---

## 7. Discovery widgets — `lib/widgets/marketplace/discovery/`

All widgets: `ConsumerWidget`/`StatelessWidget`, `AzSurfaceSpec` for radius, no borders, no blur, `RepaintBoundary` on image-bearing rails. Text tiers from `AzText`. Horizontal rails use `ListView.separated(scrollDirection: Axis.horizontal)` inside a fixed `SizedBox(height: …)` — horizontal inside vertical `CustomScrollView` is the one allowed nesting (different axes).

### 7.1 `discovery_header.dart`

```dart
class DiscoveryHeader extends ConsumerWidget {
  final VoidCallback onSearchTap; // focuses the nav pill search
  const DiscoveryHeader({super.key, required this.onSearchTap});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final nearby = ref.watch(discoverySnapshotProvider.select((s) => s.distanceKmByBizId != null));
    return Padding(
      padding: const EdgeInsets.fromLTRB(AzSpace.xl, AzSpace.sm, AzSpace.xl, AzSpace.md),
      child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
        Text('Discover', key: const ValueKey('marketplace_portal_title'), style: AzText.titleXl.copyWith(color: colors.textPrimary)),
        const SizedBox(height: AzSpace.xs),
        Text(nearby ? 'Around you, on Azaman.' : 'Everything local, on Azaman.',
            key: const ValueKey('marketplace_portal_tagline'),
            style: AzText.body.copyWith(color: colors.textSecondary), maxLines: 2),
        const SizedBox(height: AzSpace.lg),
        Semantics(
          button: true, label: 'Search the marketplace',
          child: ScaleTap(
            onTap: onSearchTap,
            child: Container(
              height: 48,
              padding: const EdgeInsets.symmetric(horizontal: AzSpace.lg),
              decoration: BoxDecoration(color: colors.softSurface, borderRadius: BorderRadius.circular(AzRadius.pill)),
              child: Row(children: [
                Icon(HugeIconsSolid.search01, size: 18, color: colors.textTertiary),
                const SizedBox(width: AzSpace.md),
                Expanded(child: Text('What are you looking for?', style: AzText.body.copyWith(color: colors.textTertiary))),
              ]),
            ),
          ),
        ),
      ]),
    );
  }
}
```

Keep the existing `ValueKey`s (`marketplace_portal_title`, `marketplace_portal_tagline`) so current tests keep passing; update any test asserting the literal 'Marketplace' text.

### 7.2 `intent_rail.dart`

```dart
class DiscoveryIntentItem {
  final AzIntent intent;
  final String label;
  final IconData icon;
  final String? worldWire;     // Eat/Shop/Ride/Stay
  final DiscoverySignal? signal; // Nearby
  final bool savesOnly;
  final bool recentOnly;
  const DiscoveryIntentItem({required this.intent, required this.label, required this.icon, this.worldWire, this.signal, this.savesOnly = false, this.recentOnly = false});
}

const kDiscoveryIntents = [
  DiscoveryIntentItem(intent: AzIntent.order, label: 'Eat', icon: HugeIconsSolid.restaurant01, worldWire: 'FOOD_BEVERAGE'),
  DiscoveryIntentItem(intent: AzIntent.open, label: 'Shop', icon: HugeIconsSolid.shoppingBag01, worldWire: 'RETAIL'),
  DiscoveryIntentItem(intent: AzIntent.book, label: 'Ride', icon: HugeIconsSolid.bus01, worldWire: 'LOGISTICS'),
  DiscoveryIntentItem(intent: AzIntent.book, label: 'Stay', icon: HugeIconsSolid.hotel01, worldWire: 'HOSPITALITY'),
  DiscoveryIntentItem(intent: AzIntent.discover, label: 'Nearby', icon: HugeIconsSolid.location01, signal: DiscoverySignal.nearby),
  DiscoveryIntentItem(intent: AzIntent.save, label: 'My saves', icon: HugeIconsSolid.bookmark01, savesOnly: true),
  DiscoveryIntentItem(intent: AzIntent.discover, label: 'Recent', icon: HugeIconsSolid.clock01, recentOnly: true),
];

class IntentRail extends ConsumerWidget {
  final void Function(DiscoveryIntentItem item) onSelect;
  const IntentRail({super.key, required this.onSelect});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final supported = ref.watch(marketplaceDiscoveryGatewayProvider).supportedSignals;
    final hasSaves = ref.watch(savedBusinessesProvider.select((s) => s.isNotEmpty));
    final hasRecent = ref.watch(worldMemoryProvider.select((m) => m.values.any((w) => w.isFresh(DateTime.now()))));
    final items = kDiscoveryIntents.where((i) {
      if (i.signal != null && !supported.contains(i.signal)) return false;
      if (i.savesOnly && !hasSaves) return false;
      if (i.recentOnly && !hasRecent) return false;
      return true;
    }).toList();
    return SizedBox(
      height: 40,
      child: ListView.separated(
        key: const ValueKey('marketplace_intent_rail'),
        scrollDirection: Axis.horizontal,
        padding: const EdgeInsets.symmetric(horizontal: AzSpace.xl),
        itemCount: items.length,
        separatorBuilder: (_, __) => const SizedBox(width: AzSpace.sm),
        itemBuilder: (_, i) => _IntentChip(item: items[i], colors: colors, onTap: () { AzamanHaptics.selection(); onSelect(items[i]); }),
      ),
    );
  }
}

class _IntentChip extends StatelessWidget {
  ...
  // Container(radius pill, color softSurface, Row(icon 16, SizedBox 6, Text(label, AzText.label)))
}
```

Icon names: verify against `hugeicons_pro` exports; substitute the nearest existing glyph rather than adding a dependency.

### 7.3 `world_deck.dart` — replaces `_worldsSection`

Each card shows: world label, a real count ("12 places"), a real signal line when supported (`"4 open now"` / `"2 within 1 km"`), and the vertical accent from `lib/theme/az_vertical_accent.dart`. No imagery is fabricated: if the catalog has businesses with `coverImageUrl`, the card uses up to 3 of them as a muted collage; otherwise a flat accent fill.

```dart
class WorldCardModel {
  final BusinessCategory category;
  final int count;
  final int? openNow;      // null when hours unsupported
  final int? withinKm;     // null when location unsupported
  final List<String> coverUrls; // ≤ 3, real
  const WorldCardModel({required this.category, required this.count, this.openNow, this.withinKm, this.coverUrls = const []});
}

List<WorldCardModel> buildWorldCards(DiscoverySnapshot s, Set<DiscoverySignal> supported) {
  bool inWorld(BusinessProfile b, String wire) =>
      wire == 'HOSPITALITY' ? (b.category == 'HOSPITALITY' || b.category == 'REAL_ESTATE') : b.category == wire;
  return BusinessCategories.values.map((cat) {
    final members = s.catalog.where((b) => inWorld(b, cat.wire)).toList();
    final open = supported.contains(DiscoverySignal.openNow)
        ? members.where((b) => b.openStateAt(s.now) == OpenState.open).length
        : null;
    final near = (supported.contains(DiscoverySignal.nearby) && s.distanceKmByBizId != null)
        ? members.where((b) => (s.distanceKmByBizId![b.bizId] ?? 99) <= 1.0).length
        : null;
    return WorldCardModel(
      category: cat, count: members.length, openNow: open, withinKm: near,
      coverUrls: members.map((b) => b.coverImageUrl).whereType<String>().take(3).toList(),
    );
  }).toList();
}
```

Widget: a 2×2 grid (`GridView.count` with `shrinkWrap: true, physics: NeverScrollableScrollPhysics()` inside the sliver adapter), card radius `AzRadius.lg`, `RepaintBoundary` per card, `AzIdentityMorph` **not** used here (worlds are not identities). Tap → `_enterExplore(cat.wire)`. Keep `ValueKey('marketplace_worlds_header')` on the section title "Choose your world".

Signal line rule (brief §3.4): `if (model.openNow != null && model.openNow! > 0) '${model.openNow} open now' else if (model.withinKm != null && model.withinKm! > 0) '${model.withinKm} within 1 km' else '${model.count} places'`. A world with `count == 0` renders the card disabled (`Opacity 0.5`, no tap) — not hidden, so the four-world mental model stays stable.

### 7.4 `utility_rail.dart`

Filters that re-run the existing `businessSearchProvider.search(...)`/sort. Only render chips whose signal is supported; "Deals" is only added when a `MarketplaceOffer` model exists in the repo (it does not in the snapshot — omit).

```dart
enum UtilityFilter { nearMe, openNow, topRated, saved, newest }

class UtilityRail extends ConsumerWidget {
  final UtilityFilter? active;
  final ValueChanged<UtilityFilter?> onChanged;
  ...
  // Map UtilityFilter → DiscoverySignal to decide visibility; 'saved' visible when savedBizIds not empty.
}
```

Behavior in the screen: `nearMe` → `_fireNearby()` + `_viewMode = map`; `openNow`/`topRated`/`saved` → client-side predicate over `businessSearchProvider.results` applied in the explore list builder (keep server query untouched); `newest` → only if the API exposes `sort=newest` (check `BusinessService.searchBusinesses` params; otherwise omit the chip).

### 7.5 `merchant_stories_row.dart`

Reuse `MarketplaceExpandedStories(onOpenBusiness:, onBrowsePressed:)` from `lib/widgets/marketplace/marketplace_status_rail.dart`. Wrap in a titled section "From the shops you follow". `onOpenBusiness(businessProfileId)` → the existing business route. Story→store continuity is handled in 05 §7 (viewer tray). Hidden when the followed set is empty.

### 7.6 `local_pulse.dart`

```dart
class LocalPulseModel {
  final int? openNow;         // null → row hidden
  final int? nearby;          // within 2 km
  const LocalPulseModel({this.openNow, this.nearby});
  bool get isEmpty => (openNow ?? 0) == 0 && (nearby ?? 0) == 0;
}
```

Derived from `DiscoverySnapshot` in a `Provider<LocalPulseModel>`. Renders two compact lines max ("18 places open now", "6 within 2 km"), each tappable → applies the matching `UtilityFilter`. `departingSoon`/`availableTonight` are **not** rendered until the gateway reports them supported (they need trip/room data on the discovery surface — leave `// TODO(gateway)`).

### 7.7 `resume_card.dart`

```dart
class ResumeCard extends ConsumerWidget {
  const ResumeCard({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final intent = ref.watch(marketplaceResumeProvider);
    if (intent == null) return const SizedBox.shrink();
    final colors = ref.watch(themeProvider).colors;
    return Padding(
      padding: const EdgeInsets.fromLTRB(AzSpace.xl, 0, AzSpace.xl, AzSpace.lg),
      child: ScaleTap(
        onTap: () => _resume(context, ref, intent),
        child: Container(
          key: const ValueKey('marketplace_resume_card'),
          padding: const EdgeInsets.all(AzSpace.lg),
          decoration: BoxDecoration(color: colors.card, borderRadius: BorderRadius.circular(AzRadius.lg)),
          child: Row(children: [
            Icon(intent.kind == ResumeKind.cart ? HugeIconsSolid.shoppingCart01 : HugeIconsSolid.search01, color: colors.accent),
            const SizedBox(width: AzSpace.md),
            Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Text('Pick up where you left off', style: AzText.caption.copyWith(color: colors.textTertiary)),
              Text(intent.title, style: AzText.title.copyWith(color: colors.textPrimary), maxLines: 2),
              Text(intent.subtitle, style: AzText.bodyS.copyWith(color: colors.textSecondary)),
            ])),
            Icon(Icons.chevron_right_rounded, color: colors.textTertiary),
          ]),
        ),
      ),
    );
  }

  void _resume(BuildContext context, WidgetRef ref, ResumeIntent intent) {
    switch (intent.kind) {
      case ResumeKind.cart:
        context.push(AzRoutes.cart);                // NEW: `AzRoutes.cart = '/marketplace/cart'` + `AzRouteNames.cart` — no cart route exists on main; register it in route_registry.dart pointing at the EXISTING `lib/screens/marketplace/cart_screen.dart`
      case ResumeKind.worldSearch:
        // handled by the screen: _enterExplore(intent.worldWire!) which recalls the memory
        ResumeIntentNotification(intent).dispatch(context);
      case ResumeKind.savedHotel:
        context.push(AzRoutes.businessProfile(intent.bizId!));   // EXISTING: route_registry.dart `businessProfile(String bizId) => '/business/$bizId'`
    }
  }
}

class ResumeIntentNotification extends Notification {
  final ResumeIntent intent;
  const ResumeIntentNotification(this.intent);
}
```

`MarketplaceHomeScreen` wraps the portal `CustomScrollView` in `NotificationListener<ResumeIntentNotification>` and calls `_enterExplore(n.intent.worldWire!)`.

### 7.8 `merchant_card.dart` — reassessed `BusinessCard` (brief §5.4)

Not a replacement of `CollapsibleBusinessBar` in the explore list (that stays). `MerchantCard` is the new **portal/featured** card answering the five questions with ≤ 5 elements:

```dart
class MerchantCard extends ConsumerWidget {
  final BusinessProfile business;
  final double? distanceKm;
  final VoidCallback onOpen;
  const MerchantCard({super.key, required this.business, this.distanceKm, required this.onOpen});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final travel = AzMotion.of(context).travel;
    final profile = MarketplaceExperienceCatalog.fromCategory(business.category);
    final open = ref.watch(discoveryClockProvider.select((now) => business.openStateAt(now))); // clock from the discovery notifier; never DateTime.now() in build
    final saved = ref.watch(savedBusinessesProvider.select((s) => s.contains(business.bizId)));

    return RepaintBoundary(
      child: ScaleTap(
        onTap: onOpen,
        child: Container(
          width: 220,
          decoration: BoxDecoration(color: colors.card, borderRadius: BorderRadius.circular(AzRadius.lg)),
          clipBehavior: Clip.antiAlias,
          child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
            // 1. what is this place — cover + logo identity (logo is the Hero)
            SizedBox(height: 110, child: Stack(fit: StackFit.expand, children: [
              if (business.coverImageUrl != null) AzamanNetworkImage(imageUrl: business.coverImageUrl!, fit: BoxFit.cover)
              else Container(color: colors.accent.withValues(alpha: .18)), // per-vertical tint: read the accent exposed by AzVerticalAccentScope (lib/theme/az_vertical_accent.dart) if the card is rendered inside one
              Positioned(left: AzSpace.md, bottom: -16, child: AzIdentityMorph(
                tag: AzIdentityTag.business(business.bizId), travel: travel,
                child: ChatAvatar(imageUrl: business.logoUrl, name: business.businessName, size: 40),
              )),
            ])),
            const SizedBox(height: AzSpace.xl),
            Padding(padding: const EdgeInsets.fromLTRB(AzSpace.md, 0, AzSpace.md, AzSpace.md),
              child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
                // 2. why care — name + trust
                Row(children: [
                  Expanded(child: Text(business.businessName, style: AzText.title.copyWith(color: colors.textPrimary), maxLines: 1, overflow: TextOverflow.ellipsis)),
                  TrustMark(business: business),
                ]),
                // 3. relevant now — open state / distance (only real)
                Text([
                  if (open == OpenState.open) 'Open now',
                  if (open == OpenState.closed) 'Closed',
                  if (distanceKm != null) '${distanceKm!.toStringAsFixed(distanceKm! < 10 ? 1 : 0)} km',
                ].join(' · '), style: AzText.bodyS.copyWith(color: open == OpenState.open ? colors.success : colors.textSecondary)),
                const SizedBox(height: AzSpace.sm),
                // 4. what can I do — one primary intent from the vertical catalog
                Row(children: [
                  Expanded(child: Text(profile.primaryActionLabel, style: AzText.label.copyWith(color: colors.accent))),
                  // 5. save
                  Icon(saved ? HugeIconsSolid.bookmark02 : HugeIconsSolid.bookmark01, size: 18, color: saved ? colors.accent : colors.textTertiary),
                ]),
              ])),
          ]),
        ),
      ),
    );
  }
}
```

(`AzamanColors.success` exists in `lib/providers/theme_provider.dart:565`.) Replace `_featuredRail`'s item builder with `MerchantCard`.

### 7.9 `trust_mark.dart` — Azaman trust language (brief §7.5)

One compact component used in cards, store identity, story business tray, invoice headers:

```dart
enum TrustLevel { verified, kybPending, none }

class TrustMark extends ConsumerWidget {
  final BusinessProfile business;
  final bool compact;
  const TrustMark({super.key, required this.business, this.compact = true});

  TrustLevel get level => business.isVerified
      ? TrustLevel.verified
      : (business.kybStatus == 'PENDING' ? TrustLevel.kybPending : TrustLevel.none);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    if (level == TrustLevel.none) return const SizedBox.shrink();
    final colors = ref.watch(themeProvider).colors;
    final icon = level == TrustLevel.verified ? HugeIconsSolid.checkmarkBadge01 : HugeIconsSolid.clock01;
    final label = level == TrustLevel.verified ? 'Verified' : 'Verification pending';
    return Semantics(
      label: '$label business',
      child: compact
          ? Icon(icon, size: 16, color: level == TrustLevel.verified ? colors.accent : colors.textTertiary)
          : Row(mainAxisSize: MainAxisSize.min, children: [Icon(icon, size: 14), const SizedBox(width: 4), Text(label, style: AzText.caption)]),
    );
  }
}
```

Rule: exactly one `TrustMark` per surface. Booking/payment/cancellation states keep their existing presentation; they are not restyled into badges.

---

## 8. `_portalBody` rewrite (anchor and replacement)

Anchor in `marketplace_home_screen.dart` ~L482:

```dart
  Widget _portalBody(AzamanColors colors) {
    return Column(
      key: const ValueKey('marketplace_portal_body'),
      children: [
        _portalHero(colors),
        ...
        _portalStories(colors),
        _portalFeatured(colors),
        _portalExploreAll(colors),
      ],
    );
  }
```

Replacement:

```dart
  Widget _portalBody(AzamanColors colors) {
    final snapshot = ref.watch(discoverySnapshotProvider);
    final supported = ref.watch(marketplaceDiscoveryGatewayProvider).supportedSignals;
    return NotificationListener<ResumeIntentNotification>(
      onNotification: (n) { if (n.intent.worldWire != null) _enterExplore(n.intent.worldWire!); return true; },
      child: CustomScrollView(
        key: const ValueKey('marketplace_portal_body'),
        controller: _portalScroll,
        physics: const AlwaysScrollableScrollPhysics(parent: ClampingScrollPhysics()),
        slivers: [
          SliverToBoxAdapter(child: DiscoveryHeader(onSearchTap: _focusNavSearch)),
          const SliverToBoxAdapter(child: ResumeCard()),
          SliverToBoxAdapter(child: IntentRail(onSelect: _onIntent)),
          const SliverToBoxAdapter(child: SizedBox(height: AzSpace.xxl)),
          SliverToBoxAdapter(child: WorldDeck(cards: buildWorldCards(snapshot, supported), onEnter: _enterExplore)),
          const SliverToBoxAdapter(child: SizedBox(height: AzSpace.xl)),
          SliverToBoxAdapter(child: UtilityRail(active: _utility, onChanged: _onUtility)),
          const SliverToBoxAdapter(child: SizedBox(height: AzSpace.xl)),
          SliverToBoxAdapter(child: _portalStories(colors)),           // existing, now wrapping MarketplaceExpandedStories
          const SliverToBoxAdapter(child: LocalPulse()),
          SliverToBoxAdapter(child: _portalFeatured(colors)),          // existing, items → MerchantCard
          SliverToBoxAdapter(child: _portalExploreAll(colors)),
          SliverPadding(padding: AzSpace.navClearance),
        ],
      ),
    );
  }

  void _focusNavSearch() {
    ref.read(marketplaceSearchProvider.notifier).focus(true);
    // Today the explore TextField's FocusNode observes `focused` via MarketplaceSearchBinding;
    // a future in-pill field binds the same way (§4.1). No nav code is touched here.
  }

  void _onIntent(DiscoveryIntentItem item) {
    if (item.worldWire != null) return _enterExplore(item.worldWire!);
    if (item.signal == DiscoverySignal.nearby) return _onUtility(UtilityFilter.nearMe);
    if (item.savesOnly) return context.push(AzRoutes.savedBusinesses); // EXISTING: '/biz/saved'
    if (item.recentOnly) {
      final fresh = ref.read(worldMemoryProvider).entries.where((e) => e.value.isFresh(DateTime.now())).toList()
        ..sort((a, b) => b.value.touchedAt.compareTo(a.value.touchedAt));
      if (fresh.isNotEmpty) _enterExplore(fresh.first.key);
    }
  }
```

`_portalHero` is deleted (replaced by `DiscoveryHeader`); `_worldsSection` is deleted (replaced by `WorldDeck`). The outer `RefreshIndicator` (anchor `onRefresh: () => _viewMode == _ViewMode.list ? _refresh() : _fireNearby()` ~L686) stays and wraps the `CustomScrollView` (it requires the scrollable to be its direct descendant — the `NotificationListener` is transparent to it).

---

## 9. Store search scope hand-off

**EXISTING:** the store page is `MarketStorefrontShell` hosted by `lib/screens/marketplace/business_profile_screen.dart:1172` (`storeNavContextProvider` does not exist). In `_BusinessProfileScreenState.initState` (after the business resolves) call:

```dart
ref.read(marketplaceSearchProvider.notifier).setScope(
  MarketplaceSearchScope.store, worldWire: business.category, storeBizId: business.bizId);
```

and in `dispose` → `setScope(MarketplaceSearchScope.world, worldWire: business.category)`. Store-scoped text filters the loaded catalog (03 §1.3) — no network. The shell itself is not modified.

---

## 10. Tests (M1)

- `test/providers/marketplace_search_provider_test.dart`: submit persists recent (mock `SharedPreferences.setMockInitialValues`), suggestions below 2 chars = recent, scope change clears text, store scope submit does not call search.
- `test/providers/marketplace_world_memory_test.dart`: remember/recall/expiry.
- `test/providers/marketplace_resume_provider_test.dart`: cart present → cart intent; checking-out cart → not shown; fresh memory → worldSearch; nothing → null.
- `test/utils/business_hours_test.dart`.
- `test/widgets/marketplace/world_deck_test.dart`: counts, zero-count disabled, signals hidden when unsupported (`ProviderScope` override of the gateway with an empty `supportedSignals`).
- `test/widgets/marketplace/merchant_card_test.dart`: no `TrustMark` when unverified; open-now line only when hours present.
- Golden: `marketplace_portal` state 1–3 from brief §26 via `pumpGoldenSurface` with a 6-business fixture.
- Update `test/marketplace_experience_scope_test.dart` / any test asserting the literal 'Marketplace' title → 'Discover'.

---

## 11. Acceptance (on device)

- Portal answers "what can I do / what's relevant / where to tap" above the fold at 360dp: header + resume (if any) + intent rail + first row of the deck.
- Typing in the nav pill and in explore show identical results; back clears search first, then pops store, then returns to portal.
- Returning to a world within 30 minutes restores the query (and scroll where safe).
- No card shows a number that isn't computed from loaded data; with hours/location missing the signal lines simply disappear.
- Scrolling the portal never triggers a network request (verify with the HTTP log: `businessSearchProvider` is only hit on pull-to-refresh, submit, world entry).