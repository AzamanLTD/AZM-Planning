# 08 — Companion Architecture (Additive Lane): Models, Manifest, Authenticity, Dedupe, Privacy, A11y

> **Reconciled against live code — frontend `main` `6810007` (after #135 storefront shell, #136 Add Cash/typography), backend `main`, planning `main`.**
> Every identifier below is tagged **EXISTING** (`path:line` on live main) or **NEW** (introduced by this document). Anchors were re-verified with `git grep` on the live tree; this doc supersedes the `30188ef` snapshot version in PR #60. Review points from the PR #60 review that this document resolves:

| Review point | Resolution in this document |
|---|---|
| `_isMemberOf(...) => true` placeholder | §3 — NEW `RoomMembershipResolver` seam with provider-backed + fixed allow-list implementations and tests; injected into `SocketCompanionGateway` |
| Server-originated events, ids, dedupe | EXISTING `RealtimeEventDeduper(maxEntries: 256)` (`lib/services/realtime_event_deduper.dart:13`) for both event-id and operation-id lanes; `SocketService` listener-list pattern (`socket_service.dart:67/279`) |
| Default off / no money surfaces | unchanged: `supportedKinds == {}` until the backend emits `companion_react`; flag via EXISTING `platform_config_provider.dart`; no production surface wiring in this slice |

Implements brief §23 and §24 plus the engineering requirements in handoff v3 §12. Depends on 01 (`AzSpatialMode.isCalm`, `RealtimeEventDeduper`) and 07 (room authorisation reads the group membership view). **No production surface is wired in this PR**: it ships models, gateway, manifest, the reaction layer widget, demo adapter, and tests. Wiring into send-money success in friend chat is a separate, later slice once `Azaman_Companions_Spec.md` is approved for implementation.

New files: `lib/companion/companion_config.dart`, `companion_asset_manifest.dart`, `companion_event.dart`, `companion_gateway.dart`, `companion_reaction_controller.dart`, `lib/widgets/companion/companion_avatar.dart`, `companion_reaction_layer.dart`, `lib/experience/demo/demo_companion_gateway.dart`.

Never touched by this lane: `deposit_screen.dart`, withdrawal, balance surfaces (`AzSpatialMode.transactionalConfirmation`/`focusedAction` → companions are structurally excluded, see §5).

---

## 1. Config — small IDs, no URLs

```dart
class CompanionConfig {
  final String baseId; final String skinToneId; final String hairId; final String outfitId; final String? accessoryId;
  final int assetManifestVersion;
  const CompanionConfig({required this.baseId, required this.skinToneId, required this.hairId, required this.outfitId, this.accessoryId, required this.assetManifestVersion});

  Map<String, Object?> toJson() => {'b': baseId, 's': skinToneId, 'h': hairId, 'o': outfitId, 'a': accessoryId, 'v': assetManifestVersion};
  factory CompanionConfig.fromJson(Map<String, dynamic> j) => CompanionConfig(baseId: j['b'] as String, skinToneId: j['s'] as String, hairId: j['h'] as String, outfitId: j['o'] as String, accessoryId: j['a'] as String?, assetManifestVersion: (j['v'] as num).toInt());

  /// Stable cache key for the composed static avatar.
  String get cacheKey => '$assetManifestVersion:$baseId/$skinToneId/$hairId/$outfitId/${accessoryId ?? '-'}';
}
```

Config is social profile data (v3 §12 privacy): it travels as IDs only; the client resolves IDs against the bundled manifest. No raw storage URLs in events.

---

## 2. Versioned asset manifest — bundled JSON

`assets/companion/manifest.json`:

```json
{
  "version": 1,
  "bases": {"b_01": {"layer": "assets/companion/base/b_01.png"}},
  "skinTones": {"gh_deep": {"tint": "#5A3A2A"}, "gh_warm": {"tint": "#8C5A3C"}, "ng_rich": {"tint": "#3E2A1F"}},
  "hair": {"h_locs": {"layer": "assets/companion/hair/h_locs.png"}, "h_fade": {"layer": "assets/companion/hair/h_fade.png"}},
  "outfits": {"o_kente": {"layer": "assets/companion/outfit/o_kente.png"}, "o_ankara": {"layer": "assets/companion/outfit/o_ankara.png"}},
  "accessories": {"a_beads": {"layer": "assets/companion/acc/a_beads.png"}},
  "reactions": {"money_received": "assets/companion/reactions/money_received.json", "money_sent": "assets/companion/reactions/money_sent.json", "wave": "assets/companion/reactions/wave.json"},
  "curated": [ {"b": "b_01", "s": "gh_deep", "h": "h_locs", "o": "o_kente", "a": "a_beads", "v": 1}, {"b": "b_01", "s": "ng_rich", "h": "h_fade", "o": "o_ankara", "a": null, "v": 1} ]
}
```

Loader (`companion_asset_manifest.dart`):

```dart
class CompanionAssetManifest {
  final int version; final Map<String, String> baseLayers, hairLayers, outfitLayers, accessoryLayers, reactions; final Map<String, Color> skinTints; final List<CompanionConfig> curated;
  ...
  static Future<CompanionAssetManifest> load() async => _parse(jsonDecode(await rootBundle.loadString('assets/companion/manifest.json')));

  /// The client must know before reacting whether it can render this config.
  bool supports(CompanionConfig c) => c.assetManifestVersion <= version && baseLayers.containsKey(c.baseId) && skinTints.containsKey(c.skinToneId) && hairLayers.containsKey(c.hairId) && outfitLayers.containsKey(c.outfitId) && (c.accessoryId == null || accessoryLayers.containsKey(c.accessoryId));
}
final companionManifestProvider = FutureProvider<CompanionAssetManifest>((_) => CompanionAssetManifest.load());
```

Unsupported config → render the **generic fallback** avatar (base + neutral tint), never crash, never fetch.

---

## 3. Events — authenticity and dedupe

```dart
enum CompanionReactionKind { moneyReceived, moneySent, wave }

class CompanionEvent {
  final String eventId;              // server-issued, unique
  final String sourceOperationId;    // the confirmed financial/room operation this reacts to
  final String roomId;               // friendshipId or groupId
  final int actorUserId;
  final CompanionReactionKind kind;
  final DateTime occurredAt;
  const CompanionEvent({...});
  factory CompanionEvent.fromJson(Map<String, dynamic> j) => ...; // reject if eventId or sourceOperationId missing → FormatException
}
```

Rules (v3 §12 "Event authenticity"):
1. A client **never** constructs a `CompanionEvent` for a money reaction. The server emits `companion_react` after the transfer reaches its authoritative success state (same place it emits the existing `friend_message` money card / `onInvoicePaid` etc.).
2. `wave` (non-financial) may be client-initiated via the gateway, and still receives its `eventId` from the server echo before rendering for the peer.
3. Dedupe: `RealtimeEventDeduper(maxEntries: 256).accept(event.eventId)` **and** a second deduper keyed on `'${kind}:${sourceOperationId}'` so a reconnect that re-emits with a fresh eventId cannot replay the same transfer's reaction.
4. Order: reactions are not queued across reconnects — an event older than 30 s at receipt is dropped (stale), because a reaction is a *moment*, not a notification.

### 3.1 Gateway

```dart
abstract interface class CompanionGateway {
  Set<CompanionReactionKind> get supportedKinds;              // empty today
  Stream<CompanionEvent> events(String roomId);               // authorised: subscribing to a room the user is not a member of yields nothing
  Future<AzGatewayResult<CompanionConfig?>> configOf(int userId);
  Future<AzGatewayResult<void>> saveMyConfig(CompanionConfig c);
  Future<AzGatewayResult<void>> sendWave(String roomId);
}

class SocketCompanionGateway implements CompanionGateway {
  SocketCompanionGateway(this.socket, this.membership);
  final SocketService socket;
  final RoomMembershipResolver membership;   // injected seam (§3) — never a literal `true`
  final _byEvent = RealtimeEventDeduper(); final _byOp = RealtimeEventDeduper(); // EXISTING: lib/services/realtime_event_deduper.dart:13 (maxEntries 256)

  @override Set<CompanionReactionKind> get supportedKinds => const {}; // flip when backend emits companion_react

  @override
  Stream<CompanionEvent> events(String roomId) {
    final ctrl = StreamController<CompanionEvent>();
    void handler(Map<String, dynamic> raw) {
      final e = CompanionEvent.fromJson(raw);
      if (e.roomId != roomId) return;
      if (!membership.isMemberOf(roomId)) return;                             // room authorisation, client-side belt (server is the braces)
      if (DateTime.now().difference(e.occurredAt) > const Duration(seconds: 30)) return;
      if (!_byEvent.accept(e.eventId) || !_byOp.accept('${e.kind.name}:${e.sourceOperationId}')) return;
      ctrl.add(e);
    }
    socket.onCompanionReact(handler);                                           // add to SocketService alongside the existing onX/removeX pattern
    ctrl.onCancel = () => socket.removeCompanionReactListener(handler);
    return ctrl.stream;
  }

  ...
}
```

### 3. Room authorization seam — `lib/experience/gateways/room_membership_resolver.dart` (NEW)

The reviewed draft had `_isMemberOf(roomId) => true`. That must never exist, not even in demo code. Authorization is a first-class, injected, tested seam:

```dart
abstract interface class RoomMembershipResolver {
  /// Synchronous, reads already-loaded membership state. Unknown room → false.
  bool isMemberOf(String roomId);
}

/// Room ids are namespaced so a friendship id can never collide with a group id.
/// Format: `friend:<friendshipId>` | `group:<groupId>`.
class ProviderRoomMembershipResolver implements RoomMembershipResolver {
  ProviderRoomMembershipResolver(this.ref, {required this.myUserId});
  final Ref ref; final int myUserId;

  @override
  bool isMemberOf(String roomId) {
    final sep = roomId.indexOf(':');
    if (sep <= 0) return false;
    final kind = roomId.substring(0, sep), id = roomId.substring(sep + 1);
    switch (kind) {
      case 'friend':
        // EXISTING: friendProvider.friends is List<Map> today; InboxEntry.fromFriendMap (04) is the typed boundary.
        return ref.read(friendProvider).friends.any((f) => f['id']?.toString() == id);
      case 'group':
        // EXISTING: GroupSummary.members from lib/providers/group_chat_provider.dart
        final g = ref.read(groupListProvider).valueOrNull?.firstWhereOrNull((g) => g.id == id);
        return g != null && g.members.any((m) => m.userId == myUserId);
      default:
        return false;
    }
  }
}

/// Demo: explicit allow-list, never "everything".
class FixedRoomMembershipResolver implements RoomMembershipResolver {
  const FixedRoomMembershipResolver(this.rooms); final Set<String> rooms;
  @override bool isMemberOf(String roomId) => rooms.contains(roomId);
}
```

Tests (`test/experience/room_membership_resolver_test.dart`): friend room present/absent; group room where I am / am not a member; malformed id (`''`, `'group:'`, `'x:1'`) → false; `FixedRoomMembershipResolver` rejects rooms outside its set. `SocketCompanionGateway` gets a test where an event for a non-member room is dropped **before** dedupe is consulted.

`SocketService` gains `onCompanionReact(cb)`/`removeCompanionReactListener(cb)` following the existing `_escrowListeners` list pattern (multi-listener), and the event name `companion_react` is added to its subscription set behind `kDebugMode || remoteConfig.companionEnabled` (reuse `platform_config_provider.dart` for the flag).

---

## 4. Rendering — layered 2D, cached, Lottie reactions

### 4.1 `CompanionAvatar`

```dart
class CompanionAvatar extends ConsumerWidget {
  final CompanionConfig config; final double size;
  const CompanionAvatar({super.key, required this.config, this.size = 72});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final manifest = ref.watch(companionManifestProvider).valueOrNull;
    if (manifest == null) return SizedBox(width: size, height: size);
    final c = manifest.supports(config) ? config : manifest.curated.first;
    return ExcludeSemantics( // decorative; the money confirmation is the accessible content
      child: RepaintBoundary(
        child: _ComposedLayers(key: ValueKey(c.cacheKey), manifest: manifest, config: c, size: size),
      ),
    );
  }
}

class _ComposedLayers extends StatelessWidget { /* Stack of Image.asset layers: base (ColorFiltered with skin tint, BlendMode.modulate) → outfit → hair → accessory. All const-friendly, cached by the image cache under the asset key. */ }
```

Memory: `Image.asset(cacheWidth: (size * dpr).round())` on every layer so the image cache holds decoded layers at display size only.

### 4.2 `CompanionReactionLayer`

```dart
class CompanionReactionController extends ChangeNotifier {
  CompanionEvent? _current; bool get isPlaying => _current != null;
  void play(CompanionEvent e) { if (_current != null) return; _current = e; notifyListeners(); }  // one at a time per companion
  void done() { _current = null; notifyListeners(); }
}

class CompanionReactionLayer extends StatelessWidget {
  final CompanionReactionController controller; final CompanionAssetManifest manifest; final Widget avatar; final double size;
  ...
  // Stack: avatar; if controller._current != null && AzMotion.of(context).travel → Lottie.asset(manifest.reactions[kind], repeat: false, onLoaded: compose duration; animate once; call controller.done() on AnimationStatus.completed)
  // Reduced motion → show a static reaction frame (Lottie `frameRate`-independent: LottieBuilder with `controller` fixed at 0.6 of duration for 900 ms via a single Future.delayed? NO — use an AnimationController(duration: MotionTokens.ambient) that fires done() on complete, and render the Lottie at a fixed progress) + AzamanHaptics.moneyLanded() once.
}
```

Caps (spec): one active companion by default, hard cap two visible per screen → `CompanionStage` widget accepts at most two children and asserts in debug. Each wrapped in `RepaintBoundary`. Lottie clips are bundled (`assets/companion/reactions/*.json`), never networked.

---

## 5. Placement rules enforced in code, not prose

```dart
/// A surface asks whether a companion may appear. Calm modes and all custody
/// surfaces return false. Called by the (future) wiring slice, never bypassed.
bool companionAllowed(AzSpatialMode mode, {required bool isCustodySurface, required bool userEnabled}) =>
    userEnabled && !isCustodySurface && !mode.isCalm && mode == AzSpatialMode.social;
```

- `userEnabled` = `settingsProvider` flag `companions_enabled` (default **off** until launch; real off switch).
- `isCustodySurface` is `true` for Deposit, Withdrawal, balance cards, Add Cash, P2P order confirmation — passed explicitly by the caller; there is no way to render a companion without answering it.
- Send-money-first: the only `CompanionReactionKind`s are `moneyReceived`/`moneySent`/`wave`; the first wiring slice targets the friend-chat money card (`chat_money_card.dart`) **after** the server emits `companion_react` for `MessageKind` money events.

---

## 6. Demo adapter — `lib/experience/demo/demo_companion_gateway.dart`

Behind `DemoGuard.enabled`: `supportedKinds = all`, `events()` emits a scripted sequence with fake-but-well-formed `eventId`/`sourceOperationId` so goldens and the reaction layer can be exercised. Never bound in production providers; only via `ProviderScope.overrides` in tests and the debug gallery (if the repo has one — check `lib/screens` for a dev gallery; otherwise skip).

---

## 7. Tests (K1)

- `companion_config_test.dart`: json round trip; cacheKey stability.
- `companion_asset_manifest_test.dart`: `supports()` true/false per missing id; future version unsupported.
- `companion_gateway_dedupe_test.dart`: duplicate eventId dropped; same op with new eventId dropped; stale event dropped; wrong room dropped; non-member dropped.
- `companion_reaction_controller_test.dart`: one at a time; `done()` releases.
- Widget: `CompanionAvatar` excluded from semantics; reduced motion → no Lottie ticking, haptic once, `done()` after `MotionTokens.ambient`.
- Golden: two curated configs side by side (static).

---

## 8. Report lines for this slice

The PR for K1 must state explicitly: no user-visible change; `companions_enabled` default false; `supportedKinds` empty until `companion_react` exists server-side; list of backend dependencies (event emission on transfer success with `eventId` + `sourceOperationId`, room-scoped delivery, `GET/PUT /users/me/companion`, `GET /users/:id/companion`).