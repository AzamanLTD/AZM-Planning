# 01 — Foundation: Audit Map, Experience Model, Motion Contracts, Gateways, Tests

> **Reconciled against live code — frontend `main` `6810007` (after #135 storefront shell, #136 Add Cash/typography), backend `main`, planning `main`.**
> Every identifier below is tagged **EXISTING** (`path:line` on live main) or **NEW** (introduced by this document). Anchors were re-verified with `git grep` on the live tree; this doc supersedes the `30188ef` snapshot version in PR #60. Review points from the PR #60 review that this document resolves:

| Review point | Resolution in this document |
|---|---|
| Anchors / invented names | Everything in this doc is NEW under `lib/experience/**`; the only EXISTING dependencies are `AzMotion.of(context).travel` (`lib/theme/az_motion.dart`), `MotionTokens`, `AzSheetGeometry`/`AzSheetWeight`, `AzamanHaptics`, `AzRadius` (`lg`, not the nonexistent `card`), `pumpGoldenSurface` (`test/goldens/golden_harness.dart`) |
| Capability truth | `AzGatewayResult` / `AzUnsupported` carry the exact missing endpoint; demo adapters are behind `DemoGuard` and never reachable in release |
| Companion authorization | `lib/experience/gateways/room_membership_resolver.dart` is added to the gateway list here (full contract in 08 §3) so 07/08 share one seam |

Implements brief §3 (principles), §4 (Deliverable A), §20 (motion language), §21 (guardrails), §22 (state/data architecture), §25/§26 (test strategy conventions). Everything here is **additive**: no existing screen changes in this document. Later documents consume these primitives.

---

## 1. Phase 0 audit — architecture map to commit as `docs/ARCHITECTURE_MAP.md`

The agent must verify each row on the live repo before writing the map. Findings from the snapshot:

### 1.1 Canonical components (keep, extend)

| Area | Canonical | Notes |
|------|-----------|-------|
| Chat tab root | `lib/screens/friends/friends_hub_screen.dart` `FriendsHubScreen` (mounted from `lib/main.dart` tab switch ~L556) | Owns inbox, story rail, friend search, requests sheet |
| Personal chat | `lib/screens/friends/friend_chat_screen.dart` | `premiumChatProvider(ChatContextParams(context: ChatContext.friend, contextId: friendshipId))` |
| Group chat | `lib/screens/group_chat/group_chat_screen.dart` + `group_profile_screen.dart` | `groupDetailProvider`, `susuInitiationStatusProvider(groupId)`, `groupActionsProvider` |
| Chat engine | `lib/providers/premium_chat_provider.dart` `PremiumChatNotifier` | Socket events `friend_message`, `new_group_message`, `message_ack`, `reaction_updated`, … |
| Stories | `lib/providers/story_provider.dart` `storyFeedProvider`, `lib/models/story_model.dart` `StoryGroup/StoryItem`, `lib/widgets/story_ring.dart` | Endpoints `/stories/feed`, `/stories/:id/view|boost|reply`, `/stories` multipart, `/stories/highlights`, `/stories/analytics/business/:id` |
| Susu | `lib/models/susu_model.dart`, `lib/providers/susu_provider.dart` (`susuDetailV2Provider`, `susuCyclesProvider`, `susuMembersProvider`, `susuSuppliedRateProvider`), `lib/services/susu_service.dart` | Group ↔ Susu link via `GroupSummary.susuGroupId/susuStatus` |
| Marketplace home | `lib/screens/marketplace/marketplace_home_screen.dart` (`_MarketplaceHomeMode.portal/explore`) | `businessSearchProvider`, `nearbySearchProvider`, `featuredBusinessesProvider`, `savedBusinessesProvider` |
| Vertical grammar | `lib/marketplace/experience/marketplace_experience_capabilities.dart` (`MarketplaceExperienceCatalog`), `lib/marketplace/experiences/marketplace_experience_blueprint.dart`, `lib/widgets/marketplace/marketplace_vertical_experience_stage.dart` | Wires: `LOGISTICS`=Transit, `FOOD_BEVERAGE`=Restaurants, `HOSPITALITY`(+legacy `REAL_ESTATE`)=Hotels, `RETAIL`=Retail |
| Flip-book menu | `lib/widgets/marketplace/restaurant_native_menu_journey.dart` `RestaurantNativeMenuJourney` + `restaurant_menu_journey_adapter.dart` | Takes `sections: List<CatalogSection>`, `dishesById: Map<String, RestaurantDish>` |
| Sheets | `lib/theme/az_sheet.dart` `AzSheetWeight {whisper, panel, stage}`, `AzSheetGeometry`, `AzamanSheet.showPanel(context, builder: (ctx, scrollController))` | Use for every new bottom sheet |
| Motion | `lib/theme/motion_tokens.dart` `MotionTokens`, `lib/theme/az_motion.dart` `AzMotion` | `AzMotion.of(ctx).travel` is the reduced-motion gate |
| Haptics | `lib/utils/azaman_haptics.dart` `AzamanHaptics` (`selection`, `threshold`, `commit`, `confirm`, `nav`, `moneyLanded`, …) | Never call `HapticFeedback.*` directly in new code |
| Realtime | `lib/services/socket_service.dart` `SocketService` (callback registration), `lib/services/realtime_event_deduper.dart` `RealtimeEventDeduper.accept(eventId)` | Reuse the deduper for story reactions & companion events |
| Placement | `lib/widgets/liquid/liquid_placement.dart` `solvePanel(...)`, `LiquidSafeArea`, `PanelPlacement` | Anchored popovers |
| Goldens | `test/goldens/golden_harness.dart` `pumpGoldenSurface`, `loadGoldenFonts` | Pinned size/DPR/Inter/disableAnimations |

### 1.2 Duplicates / dead / conflicting (resolve in F1 or note as migration)

| Finding | Evidence | Action |
|---------|----------|--------|
| `lib/screens/messages_hub_screen.dart` `MessagesHubScreen` duplicates the inbox + story rail (`storyFeedProvider` + `StoryRing` ~L503–560) | Only referenced from `lib/router/app_router.dart:279`; tab root is `FriendsHubScreen` | Mark deprecated in the map. Do **not** delete in F1; C1 re-points the route to `FriendsHubScreen` once the hub rewrite lands, then a follow-up removes it |
| Story rail rendered three ways | `friends_hub_screen.dart` horizontal `ListView.builder`, `messages_hub_screen.dart`, `widgets/marketplace/marketplace_status_rail.dart` (`MarketplaceExpandedStories`) | C1 introduces `StoryRailStrip` (04 §3) used by chat; 02 reuses `MarketplaceExpandedStories` for merchant stories |
| Three-dot in personal chat is a placeholder | `friend_chat_screen.dart:606` `IconButton(icon: Icon(Icons.more_vert), onPressed: _openChatProfile)` | 07 §5 replaces with `ChatActionsSheet` |
| `Inbox` header hardcodes `fontSize: 28, w800` | `friends_hub_screen.dart` ~L330 | C1 → `AzText.titleXl` |
| Marketplace has its own `_MarketplaceHomeMode` enum + `_searchExpanded` local search | `marketplace_home_screen.dart:63, 113` | 02 §4 introduces `marketplaceSearchProvider`; the nav pill (v3 B) and the explore screen read the same state. `_MarketplaceHomeMode` stays (it is a *screen* mode, not nav state) |
| Story viewer is single-group | `story_viewer_screen.dart` has one `_progress` controller and no horizontal `PageView` | 05 rebuilds |
| Story editor keeps one stroke (`List<Offset> _drawPoints`) and ad-hoc `_OverlayItem` | `story_editor_screen.dart:40,101` | 06 introduces `StoryScene` document |
| `Column(... Expanded(ListView))` hub body means the story rail is outside the scroll owner | `friends_hub_screen.dart` build | 04 converts to one `CustomScrollView` |
| `StoryCameraScreen` simulates a preview (`image_picker` only, no `camera` dep) | `pubspec.yaml` has no `camera` | 06 §1 is an explicit decision point |

### 1.3 Scroll-owner inventory (to be verified)

| Screen | Vertical owners today | Target |
|--------|----------------------|--------|
| FriendsHub | `Column` → fixed rail + `ListView.separated` | one `CustomScrollView` |
| FriendChat | `ListView.builder(reverse: true)` | unchanged; search jumps use `Scrollable.ensureVisible` |
| BusinessProfile | EXISTING `MarketStorefrontShell` (#135): `DraggableScrollableSheet` with `MarketStorefrontSnaps` | unchanged — 03 adds content via `productsBuilder` only |
| MarketplaceHome | `RefreshIndicator` → portal `Column` / explore list | 02: portal becomes a `CustomScrollView` |
| StoryViewer | none (Stack) | 05: horizontal `PageView` + vertical drag owned by the page |

---

## 2. Spatial modes, intents, surfaces — `lib/experience/`

These are vocabulary types. They are **not** a new state machine; existing screens map their own state onto them for semantics, analytics and motion defaults.

### 2.1 `lib/experience/az_spatial_mode.dart`

```dart
/// Where the user *is*. Screens expose their current mode so that motion,
/// semantics announcements and analytics speak one language.
enum AzSpatialMode {
  /// Destination entry (Marketplace portal, Chat inbox at rest).
  portal,
  /// Browsing a catalogue (explore list/map, store shopping depth).
  explore,
  /// Quick peek without leaving the list (dossier sheet, quick peek).
  preview,
  /// Full detail of one entity (product, room, trip).
  detail,
  /// Immersive media (story viewer, media viewer).
  fullScreenMedia,
  /// Cart/tray active, purchase affordances docked.
  shopping,
  /// Keyboard-bound or form-bound task (search, compose, Add Cash).
  focusedAction,
  /// Conversation / group / Susu context.
  social,
  /// Money is about to move or has moved; UI must stay calm.
  transactionalConfirmation;

  /// Calm modes forbid expressive motion and companion reactions.
  bool get isCalm =>
      this == AzSpatialMode.transactionalConfirmation ||
      this == AzSpatialMode.focusedAction;
}
```

### 2.2 `lib/experience/az_intent.dart`

```dart
/// What the user is trying to *do*. Rails, menus and CTAs are built from
/// intents so that labels, icons and haptics are consistent app-wide.
enum AzIntent {
  discover, search, compare, save, open, book, order, pay, chat, join,
  react, reply, share, report, mute, block,
}

extension AzIntentPresentation on AzIntent {
  bool get isDestructive => this == AzIntent.report || this == AzIntent.block;

  /// Haptic that fires when the intent *commits* (not when it is offered).
  Future<void> commitHaptic() => switch (this) {
        AzIntent.pay || AzIntent.order || AzIntent.book => AzamanHaptics.commit(),
        AzIntent.save || AzIntent.react || AzIntent.mute => AzamanHaptics.selection(),
        AzIntent.report || AzIntent.block => AzamanHaptics.warn(),
        _ => AzamanHaptics.selection(),
      };
}
```

(`import 'package:azaman/utils/azaman_haptics.dart';`)

### 2.3 `lib/experience/az_surface.dart`

```dart
/// Surface taxonomy. A visual component declares exactly one kind; the kind
/// pins its styling contract so two screens cannot restyle "a card".
enum AzSurfaceKind {
  hero, identity, contextRow, categoryRail, card, tray, sheet, inlineSearch,
  chip, noticeBoard, socialRail, statusRail, actionDock,
}

/// Styling contract per kind. Values reference tokens only.
class AzSurfaceSpec {
  final double radius;
  final bool hasBorder;
  final bool hasShadow;
  final bool hasBlur;
  const AzSurfaceSpec({
    required this.radius,
    this.hasBorder = false,
    this.hasShadow = false,
    this.hasBlur = false,
  });

  static AzSurfaceSpec of(AzSurfaceKind kind) => switch (kind) {
        AzSurfaceKind.card => const AzSurfaceSpec(radius: AzRadius.lg),
        AzSurfaceKind.noticeBoard => const AzSurfaceSpec(radius: AzRadius.md),
        AzSurfaceKind.chip => const AzSurfaceSpec(radius: AzRadius.pill),
        AzSurfaceKind.sheet => const AzSurfaceSpec(radius: AzRadius.xxl),
        AzSurfaceKind.tray => const AzSurfaceSpec(radius: AzRadius.xl, hasShadow: true),
        AzSurfaceKind.identity => const AzSurfaceSpec(radius: AzRadius.pill),
        _ => const AzSurfaceSpec(radius: AzRadius.lg),
      };
}
```

`AzRadius.lg` (22) is the token the v3 handoff Phase C asks for (`lib/theme/az_radius.dart`). If Phase C has not added it yet, add:

```dart
  /// De-facto Home/module card radius (22). Tokenised per UI Correction §7.1.
  static const double card = 22;
```

---

## 3. Motion contracts — `lib/experience/motion/`

Three reusable solvers replace per-screen magic numbers. They are plain Dart (testable without widgets) and use `MotionTokens` for durations/curves.

### 3.1 `az_snap_solver.dart` — detent snapping with velocity

Used by: story rail (04), storefront sheet verification (v3 D), retail tray (03), activity doorway (v3 C).

```dart
import 'dart:math' as math;

/// Resolves which detent a drag should settle on.
///
/// All values are in the same unit (pixels or fractions). Deterministic: the
/// same inputs always give the same target, so it is unit-testable.
class AzSnapSolver {
  final List<double> detents; // ascending
  final double commitFraction; // 0..1 of the distance between two detents
  final double flingVelocity; // units/second that forces direction

  const AzSnapSolver({
    required this.detents,
    this.commitFraction = 0.40,
    this.flingVelocity = 600,
  }) : assert(detents.length >= 2);

  double resolve(double position, double velocity) {
    // 1) Fling decides direction.
    if (velocity.abs() >= flingVelocity) {
      return velocity > 0 ? _nextAbove(position) : _nextBelow(position);
    }
    // 2) Otherwise nearest detent, biased by commitFraction from the lower one.
    final lower = _nextBelow(position);
    final upper = _nextAbove(position);
    if (lower == upper) return lower;
    final span = upper - lower;
    final t = (position - lower) / span;
    return t >= commitFraction ? upper : lower;
  }

  double _nextAbove(double p) =>
      detents.firstWhere((d) => d >= p - 1e-9, orElse: () => detents.last);
  double _nextBelow(double p) =>
      detents.lastWhere((d) => d <= p + 1e-9, orElse: () => detents.first);

  double clamp(double p) => math.max(detents.first, math.min(detents.last, p));
}
```

Test (`test/experience/az_snap_solver_test.dart`):

```dart
void main() {
  const s = AzSnapSolver(detents: [0, 120]);
  test('below commit fraction returns lower', () => expect(s.resolve(40, 0), 0));
  test('at/above commit fraction returns upper', () => expect(s.resolve(48, 0), 120));
  test('fling up overrides position', () => expect(s.resolve(10, 900), 120));
  test('fling down overrides position', () => expect(s.resolve(110, -900), 0));
  test('exact detent stays', () => expect(s.resolve(120, 0), 120));
}
```

### 3.2 `az_pull_reveal_controller.dart` — extent 0..1 with commit

Used where a hidden surface is pulled into view by the *same* gesture that owns the list (story rail is scroll-owned — see 04 — so it uses `AzSnapSolver` directly; this controller is for pull-reveals that are **not** inside a scrollable, e.g. the Home card deck or a drawer handle).

```dart
import 'package:flutter/animation.dart';
import 'package:flutter/scheduler.dart';
import 'package:azaman/theme/motion_tokens.dart';
import 'az_snap_solver.dart';

enum AzRevealState { closed, dragging, open }

class AzPullRevealController extends ChangeNotifier {
  AzPullRevealController({
    required TickerProvider vsync,
    required this.maxExtentPx,
    this.solver = const AzSnapSolver(detents: [0, 1]),
  }) : _anim = AnimationController(vsync: vsync, duration: MotionTokens.emphasized) {
    _anim.addListener(() {
      _extent = _anim.value;
      notifyListeners();
    });
  }

  final double maxExtentPx;
  final AzSnapSolver solver;
  final AnimationController _anim;

  double _extent = 0; // 0..1
  AzRevealState _state = AzRevealState.closed;

  double get extent => _extent;
  AzRevealState get state => _state;
  bool get isOpen => _state == AzRevealState.open;

  void dragStart() {
    _anim.stop();
    _state = AzRevealState.dragging;
    notifyListeners();
  }

  void dragUpdate(double deltaPx) {
    _extent = (_extent + deltaPx / maxExtentPx).clamp(0.0, 1.0);
    notifyListeners();
  }

  /// [velocityPxPerSec] positive = pulling further open.
  void dragEnd(double velocityPxPerSec, {required bool travel}) {
    final target = solver.resolve(_extent, velocityPxPerSec / maxExtentPx);
    settle(target, travel: travel);
  }

  void settle(double target, {required bool travel}) {
    _state = target >= 1 ? AzRevealState.open : AzRevealState.closed;
    if (!travel) {
      _anim.value = target; // reduced motion: jump, state still changes
      return;
    }
    _anim.animateTo(target, curve: MotionTokens.enter);
  }

  void open({required bool travel}) => settle(1, travel: travel);
  void close({required bool travel}) => settle(0, travel: travel);

  @override
  void dispose() {
    _anim.dispose();
    super.dispose();
  }
}
```

### 3.3 `az_identity_morph.dart` — the "same object, different form" contract

Used for: merchant preview → storefront identity → compact pill (02/03), story ring → viewer header (05), group avatar → group profile header (07).

```dart
import 'package:flutter/widgets.dart';

/// Hero tags are the identity carrier. One function builds them so that a
/// business keeps the same tag from a marketplace card to the store pill.
abstract final class AzIdentityTag {
  static String business(String bizId) => 'az-identity-biz-$bizId';
  static String user(int userId) => 'az-identity-user-$userId';
  static String group(String groupId) => 'az-identity-group-$groupId';
  static String story(int authorId) => 'az-identity-story-$authorId';
}

/// Wraps an avatar/logo in a [Hero] only when motion travel is allowed; with
/// reduced motion the Hero is skipped so the route change is a plain cut.
class AzIdentityMorph extends StatelessWidget {
  final String tag;
  final Widget child;
  final bool travel;
  const AzIdentityMorph({super.key, required this.tag, required this.child, required this.travel});

  @override
  Widget build(BuildContext context) {
    if (!travel) return child;
    return Hero(
      tag: tag,
      flightShuttleBuilder: (ctx, anim, dir, from, to) => to.widget,
      child: child,
    );
  }
}
```

Rule: `AzIdentityMorph` is only applied to **one** element per transition (the logo/avatar). Names cross-fade; never Hero text.

### 3.4 Motion language → tokens (brief §20)

| Word | Meaning | Implementation default |
|------|---------|------------------------|
| Glide | short spatial travel on context change | `MotionTokens.standard` + `MotionTokens.enter`, translate ≤ 24dp |
| Lift | reveal teaching a hidden surface | `AzPullRevealController`/snap, `MotionTokens.emphasized` |
| Nest | detail emerges from its launcher | `AzIdentityMorph` + `AzamanSheet.showPanel` |
| Peel | progressive media reveal | story rail header `shrinkOffset` (04), viewer dismiss scale (05) |
| Dock | controls settle compact | `AnimatedPadding/AnimatedSize` with `MotionTokens.control` |
| Seal | final lock-in after money moved | existing `AzamanHaptics.moneyLanded` + a single 350 ms scale-in, **never** a loop |

These are vocabulary, not APIs: the implementation agent uses the token pair in the table when a screen needs that motion.

---

## 4. Gateways — `lib/experience/gateways/`

Brief §22: typed interfaces where backend support is missing, real services where it exists, demo adapters isolated.

### 4.1 `az_gateway_result.dart`

```dart
/// A capability-aware result. [unsupported] lets UI *hide* an action instead
/// of rendering a dead button (brief §13 "Do not build dead buttons").
sealed class AzGatewayResult<T> {
  const AzGatewayResult();
}

class AzOk<T> extends AzGatewayResult<T> {
  final T value;
  const AzOk(this.value);
}

class AzFailed<T> extends AzGatewayResult<T> {
  final String message;
  final Object? cause;
  const AzFailed(this.message, {this.cause});
}

class AzUnsupported<T> extends AzGatewayResult<T> {
  final String reason;
  const AzUnsupported(this.reason);
}

extension AzGatewayResultX<T> on AzGatewayResult<T> {
  T? get valueOrNull => this is AzOk<T> ? (this as AzOk<T>).value : null;
  bool get isUnsupported => this is AzUnsupported<T>;
}
```

### 4.2 Capability declaration pattern

Each gateway exposes a `capabilities` set so a menu can be built from what is *actually* implemented:

```dart
abstract interface class ChatActionsGateway {
  Set<ChatActionCapability> get capabilities;
  // ... methods (see 07 §5)
}
```

Riverpod binding convention (one provider per gateway, production adapter by default, overridable in tests and demo):

```dart
final chatActionsGatewayProvider = Provider<ChatActionsGateway>((ref) {
  return HttpChatActionsGateway(ref); // real
});
```

Demo override only via `ProviderScope(overrides: [...])` behind `DemoGuard` (4.4). Never inside production provider bodies.

### 4.3 Gateway files introduced by this pack

| File | Defined in | Real adapter today |
|------|------------|--------------------|
| `story_gateway.dart` | 05 §6 / 06 §5 | partial (`/stories/*` endpoints exist); reactions/edit/privacy unsupported |
| `chat_actions_gateway.dart` | 07 §5 | search exists (`MessageActionService.searchMessages`); mute/pin/archive local-only; block/report unsupported |
| `marketplace_discovery_gateway.dart` | 02 §2 | `BusinessService.searchBusinesses/searchNearby`, `featuredBusinessesProvider` |
| `susu_membership_gateway.dart` | 07 §2 | `susuMembersProvider`, `susuInitiationStatusProvider`, invites |
| `service_flow_gateway.dart` | 03 §6 | none (seam only) |

### 4.4 `lib/experience/demo/demo_guard.dart`

```dart
import 'package:flutter/foundation.dart';

/// Single switch for preview/demo adapters. Production builds compile demo
/// code out of the dependency graph by never constructing demo adapters when
/// this is false. Default: debug-only and explicitly enabled.
abstract final class DemoGuard {
  static bool _enabled = false;
  static bool get enabled => kDebugMode && _enabled;
  @visibleForTesting
  static void enable([bool value = true]) => _enabled = value;
}
```

`lib/data/demo_mode*.dart` already exists (see `test/data/demo_mode_test.dart`); the agent should check whether it exposes a suitable flag and **reuse it** instead of this class if so. The rule is the same: demo adapters are only reachable behind one guard.

---

## 5. Repaint and rebuild guardrails (brief §21) — checklist to apply per screen

1. Any surface that re-renders on a drag tick (`shrinkOffset`, sheet extent, odometer) is wrapped in `RepaintBoundary` **at the leaf**, not around the whole screen.
2. Scroll-driven visuals read offsets through `AnimatedBuilder`/`ValueListenableBuilder` on the controller, never `setState` on the screen.
3. Provider reads inside item builders are `ref.watch(provider.select(...))` for the single field needed.
4. Futures used by `FutureBuilder` are created in `initState`/notifiers, never in `build`.
5. Every `AnimationController`/`VideoPlayerController`/`StreamSubscription` has a matching dispose; PageView children stop work when not current (05 §3).
6. No `Timer.periodic` for interaction state. The only timers allowed are semantic (story progress uses an `AnimationController`, placeholder rotation uses a ticker that is stopped on focus).

---

## 6. Test conventions for this pack

- Pure solvers (`AzSnapSolver`, scene reducers, membership derivations) get plain `test()` files under `test/experience/`.
- Widget behavior tests pump with `tester.pump(MotionTokens.emphasized)` after gestures; assert state via controller getters or `find.byKey`.
- Reduced-motion tests wrap in `MediaQuery(data: MediaQueryData(disableAnimations: true))` **inside** `MaterialApp.home` (same reason as the golden harness) and assert the semantic end state is reached after a single `pump()`.
- Golden states listed in brief §26 use `pumpGoldenSurface` with deterministic fixtures (no network images: use `Image.memory` fixtures or `CachedNetworkImage` placeholders).
- Never regenerate goldens to make a suite pass; a regeneration is its own reviewable commit.
- Key convention: `ValueKey('<screen>_<element>')` snake_case, matching the existing `marketplace_portal_body` style.