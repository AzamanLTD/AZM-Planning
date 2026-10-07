# 05 — Story Viewer: Multi-Creator Architecture, Gesture Matrix, Media Lifecycle, Interaction Layer

Implements brief §9 and the story→store / story→chat parts of §11. Depends on 01 (gateways, `AzIdentityTag`, `RealtimeEventDeduper` reuse) and 04 (`AzIdentityTag.story` Hero tag on rings).

Replaces `lib/screens/story_viewer_screen.dart` (333 lines, single group, one `_progress` controller, tap zones at 35%, long-press pause, reply field, boost badge, linked-biz tap commented out). The public opener keeps its signature so the hub, `messages_hub_screen.dart`, `business_card.dart` and `marketplace_status_rail.dart` do not change:

```dart
static Future<void> open(BuildContext context, {required List<StoryGroup> groups, required int initialGroupIndex});
```

New files (all under `lib/widgets/stories/viewer/`): `story_viewer_screen.dart` (moved), `story_playback_controller.dart`, `story_media_controller.dart`, `story_gesture_arbiter.dart`, `story_group_page.dart`, `story_progress_bar.dart`, `story_interaction_bar.dart`, `story_business_tray.dart`, `story_details_sheet.dart`. Gateway: `lib/experience/gateways/story_gateway.dart`.

---

## 1. Architecture

```
StoryViewerScreen (route, black scaffold)
└── _ViewerShell                          owns PageController (horizontal), activeGroup ValueNotifier<int>, dismiss progress
    └── StoryGestureArbiter               ONE Listener-based arbiter: decides tap / long-press / horizontal / vertical / pinch
        └── PageView.builder (NeverScrollableScrollPhysics)   horizontal drag is fed by the arbiter
            └── StoryGroupPage(group, isActive)              per creator
                ├── StoryPlaybackController (AnimationController progress, item index, pause/resume)
                ├── StoryMediaController    (image precache / VideoPlayerController lifecycle)
                ├── StoryProgressBar        (segments, reads controller via AnimatedBuilder)
                ├── header: identity (Hero AzIdentityTag.story), name, time, close
                ├── media: image (zoom transform) or VideoPlayer
                ├── caption, boost badge (existing semantics)
                ├── StoryBusinessTray?      (linkedBizId → vertical primary action)
                └── StoryInteractionBar     (reply field, reaction stickers if supported, share if supported)
```

State ownership:
- **Horizontal** (which creator) → `_ViewerShell` via `PageController` + arbiter.
- **Vertical** (dismiss / details) → `_ViewerShell` (`dismissProgress` 0..1, `detailsOpen`).
- **Playback** (which item, progress, paused) → `StoryPlaybackController` per page; only the active page runs.
- **Media** → `StoryMediaController` per page; disposed when the page is ≥ 2 away from active or the viewer closes.
- **Server truth** (viewed, reply, reaction, boost) → `StoryGateway`.

No `Timer`. Progress is an `AnimationController` with `duration = item.durationSeconds` (video: `videoController.value.duration` once known, as today).

---

## 2. Gesture matrix and the arbiter

| Input | Owner | Effect |
|-------|-------|--------|
| Tap right 65% | arbiter → playback | next item; at last item → next group (shell) |
| Tap left 35% | arbiter → playback | previous item; at first item → previous group |
| Hold ≥ 350 ms, no movement | arbiter → playback | pause; release → resume |
| Horizontal drag/fling | arbiter → PageController | previous/next creator (cube-less page slide, `MotionTokens.spatial`) |
| Vertical drag down | arbiter → shell | dismiss: page scales 1→0.85, black → transparent; release past 0.35 or fling → pop |
| Vertical drag up | arbiter → shell | opens `StoryDetailsSheet` (reply/reactions/boost/viewers for own story) when the gateway supports anything there; else no-op |
| Two pointers | arbiter → page zoom | pinch zoom 1..3× on **images only**; releases back to 1 (`MotionTokens.standard`); playback paused while zooming |
| System back | route | pop |

### 2.1 `story_gesture_arbiter.dart`

The arbiter uses a raw `Listener` and a tiny state machine so axis decisions are explicit (brief §9.2 "Gesture arbitration must be explicit"). The `PageView` has `NeverScrollableScrollPhysics`; horizontal pans are forwarded with `ScrollPosition.drag` so the page physics still produce the snap.

```dart
import 'package:flutter/gestures.dart';
import 'package:flutter/widgets.dart';

enum _Phase { idle, deciding, horizontal, vertical, pinch, holding }

class StoryGestureCallbacks {
  final void Function(bool rightSide) onTap;
  final VoidCallback onHoldStart;
  final VoidCallback onHoldEnd;
  final ValueChanged<double> onVerticalUpdate;   // +down, −up, in px
  final void Function(double velocityPxPerSec) onVerticalEnd;
  final ValueChanged<double> onPinchScale;       // absolute scale 1..∞ (caller clamps)
  final VoidCallback onPinchEnd;
  const StoryGestureCallbacks({...});
}

class StoryGestureArbiter extends StatefulWidget {
  final Widget child;
  final PageController pageController;
  final StoryGestureCallbacks callbacks;
  final double slop;                 // kTouchSlop
  final Duration holdDelay;          // 350 ms
  const StoryGestureArbiter({super.key, required this.child, required this.pageController, required this.callbacks,
      this.slop = kTouchSlop, this.holdDelay = const Duration(milliseconds: 350)});

  @override
  State<StoryGestureArbiter> createState() => _StoryGestureArbiterState();
}

class _StoryGestureArbiterState extends State<StoryGestureArbiter> {
  _Phase _phase = _Phase.idle;
  final Map<int, Offset> _pointers = {};
  Offset? _origin;
  Duration? _downTime;
  Drag? _pageDrag;
  VelocityTracker? _vt;
  double _pinchBase = 0;
  late final Ticker _holdTicker;   // not a Timer: ticks until holdDelay elapsed, then cancels itself
  double _width = 0;

  @override
  void initState() {
    super.initState();
    _holdTicker = Ticker(_onHoldTick);
  }

  void _onHoldTick(Duration elapsed) {
    if (_phase == _Phase.deciding && elapsed >= widget.holdDelay) {
      _holdTicker.stop();
      _phase = _Phase.holding;
      widget.callbacks.onHoldStart();
    }
  }

  void _down(PointerDownEvent e) {
    _pointers[e.pointer] = e.position;
    if (_pointers.length == 1) {
      _origin = e.position;
      _downTime = e.timeStamp;
      _vt = VelocityTracker.withKind(e.kind)..addPosition(e.timeStamp, e.position);
      _phase = _Phase.deciding;
      _holdTicker.start();
    } else if (_pointers.length == 2 && (_phase == _Phase.deciding || _phase == _Phase.holding || _phase == _Phase.vertical)) {
      _holdTicker.stop();
      if (_phase == _Phase.holding) widget.callbacks.onHoldEnd();
      _phase = _Phase.pinch;
      _pinchBase = _pointerDistance();
    }
  }

  void _move(PointerMoveEvent e) {
    if (!_pointers.containsKey(e.pointer)) return;
    _pointers[e.pointer] = e.position;
    _vt?.addPosition(e.timeStamp, e.position);
    switch (_phase) {
      case _Phase.deciding:
        final d = e.position - _origin!;
        if (d.distance < widget.slop) return;
        _holdTicker.stop();
        if (d.dx.abs() > d.dy.abs()) {
          _phase = _Phase.horizontal;
          _pageDrag = widget.pageController.position.drag(DragStartDetails(globalPosition: e.position), () => _pageDrag = null);
        } else {
          _phase = _Phase.vertical;
        }
        _move(e); // apply this delta
      case _Phase.horizontal:
        _pageDrag?.update(DragUpdateDetails(globalPosition: e.position, delta: Offset(e.delta.dx, 0), primaryDelta: e.delta.dx));
      case _Phase.vertical:
        widget.callbacks.onVerticalUpdate(e.delta.dy);
      case _Phase.pinch:
        if (_pointers.length == 2 && _pinchBase > 0) widget.callbacks.onPinchScale(_pointerDistance() / _pinchBase);
      case _Phase.holding:
        // Finger drift while holding keeps the hold (Instagram behaviour); large motion ends it.
        if ((e.position - _origin!).distance > widget.slop * 3) { widget.callbacks.onHoldEnd(); _phase = _Phase.vertical; }
      case _Phase.idle:
        break;
    }
  }

  void _up(PointerEvent e) {
    _pointers.remove(e.pointer);
    final velocity = _vt?.getVelocity().pixelsPerSecond ?? Offset.zero;
    switch (_phase) {
      case _Phase.deciding:
        _holdTicker.stop();
        final isTap = (e.timeStamp - (_downTime ?? e.timeStamp)) < widget.holdDelay;
        if (isTap && _origin != null) widget.callbacks.onTap(_origin!.dx > _width * 0.35);
      case _Phase.holding:
        widget.callbacks.onHoldEnd();
      case _Phase.horizontal:
        _pageDrag?.end(DragEndDetails(velocity: Velocity(pixelsPerSecond: Offset(velocity.dx, 0)), primaryVelocity: velocity.dx));
        _pageDrag = null;
      case _Phase.vertical:
        widget.callbacks.onVerticalEnd(velocity.dy);
      case _Phase.pinch:
        if (_pointers.isEmpty) widget.callbacks.onPinchEnd();
        else return; // one finger left: stay in pinch until all up (avoid accidental page flips)
      case _Phase.idle:
        break;
    }
    if (_pointers.isEmpty) { _phase = _Phase.idle; _origin = null; _vt = null; }
  }

  double _pointerDistance() {
    final p = _pointers.values.toList();
    return (p[0] - p[1]).distance;
  }

  @override
  void dispose() { _holdTicker.dispose(); _pageDrag?.cancel(); super.dispose(); }

  @override
  Widget build(BuildContext context) {
    _width = MediaQuery.sizeOf(context).width;
    return Listener(
      behavior: HitTestBehavior.opaque,
      onPointerDown: _down, onPointerMove: _move, onPointerUp: _up, onPointerCancel: _up,
      child: widget.child,
    );
  }
}
```

The reply `TextField` and buttons in the interaction bar sit **above** the arbiter in the tree (they are siblings in a `Stack`, painted later), so their own gesture recognizers win for taps on them. The arbiter only receives pointers that hit the media area.

---

## 3. Playback and media controllers

### 3.1 `story_playback_controller.dart`

```dart
class StoryPlaybackController extends ChangeNotifier {
  StoryPlaybackController({required TickerProvider vsync, required this.group, required this.onGroupComplete, required this.onGroupRewindPast})
      : _progress = AnimationController(vsync: vsync) {
    _progress.addStatusListener((s) { if (s == AnimationStatus.completed) next(); });
  }

  final StoryGroup group;
  final VoidCallback onGroupComplete;   // shell: go to next creator or pop
  final VoidCallback onGroupRewindPast; // shell: go to previous creator
  final AnimationController _progress;

  int _index = 0;
  bool _paused = false;
  bool _active = false;
  int _holds = 0; // pause reasons: hold, zoom, sheet, inactive, buffering

  int get index => _index;
  StoryItem get item => group.stories[_index];
  Animation<double> get progress => _progress;
  bool get isPaused => _holds > 0;

  void start({required int initialIndex}) {
    _index = initialIndex.clamp(0, group.stories.length - 1);
    _restart();
  }

  void setActive(bool active) { if (_active == active) return; _active = active; active ? release(PauseReason.inactive) : hold(PauseReason.inactive); }

  void setDuration(Duration d) { _progress.duration = d; if (!isPaused && _active) _progress.forward(from: _progress.value); }

  void _restart() {
    _progress.stop();
    _progress.value = 0;
    _progress.duration = Duration(seconds: item.durationSeconds.clamp(1, 60));
    notifyListeners();
    if (!isPaused && _active) _progress.forward();
  }

  void next() {
    if (_index < group.stories.length - 1) { _index++; _restart(); } else { onGroupComplete(); }
  }

  void previous() {
    if (_index > 0) { _index--; _restart(); } else { onGroupRewindPast(); }
  }

  void hold(PauseReason r) { _holds |= r.bit; _progress.stop(); notifyListeners(); }
  void release(PauseReason r) { _holds &= ~r.bit; if (!isPaused && _active) _progress.forward(); notifyListeners(); }

  @override
  void dispose() { _progress.dispose(); super.dispose(); }
}

enum PauseReason { hold(1), zoom(2), sheet(4), inactive(8), buffering(16); const PauseReason(this.bit); final int bit; }
```

Multiple pause reasons are OR-ed so releasing a hold while the details sheet is open keeps the story paused (brief: pause/resume must be deterministic).

### 3.2 `story_media_controller.dart`

```dart
class StoryMediaController extends ChangeNotifier {
  StoryMediaController(this.item);
  final StoryItem item;
  VideoPlayerController? video;
  bool ready = false;
  String? error;

  bool get isVideo => item.mediaType.toUpperCase() == 'VIDEO';

  Future<void> prepare(BuildContext context) async {
    try {
      if (isVideo) {
        video = VideoPlayerController.networkUrl(Uri.parse(item.mediaUrl));
        await video!.initialize();
        await video!.setLooping(false);
      } else {
        await precacheImage(CachedNetworkImageProvider(item.mediaUrl), context);
      }
      ready = true;
    } catch (e) {
      error = 'Could not load this story';
    }
    notifyListeners();
  }

  Future<void> play() async { if (ready && isVideo) await video!.play(); }
  Future<void> pause() async { if (isVideo) await video!.pause(); }

  @override
  void dispose() { video?.dispose(); super.dispose(); }
}
```

`StoryGroupPage` keeps a `Map<int, StoryMediaController>` for `index-1..index+1` of its own group; the shell additionally asks the next **group** page to `prepare` its first item when the user is within 0.3 of the page boundary (read `pageController.page` in a listener — cheap, no rebuild). Pages at `|i - active| ≥ 2` dispose their media controllers (`AutomaticKeepAliveClientMixin` is **not** used; `PageView` default disposal does the rest).

Video duration: when `video.value.duration` is known, `playback.setDuration(duration)`; if a buffer stall (`video.value.isBuffering`) → `hold(PauseReason.buffering)` / `release`.

---

## 4. `StoryGroupPage`

```dart
class StoryGroupPage extends ConsumerStatefulWidget {
  final StoryGroup group;
  final ValueListenable<int> activeIndex;   // shell
  final int myIndex;
  final int initialItem;
  final VoidCallback onComplete;
  final VoidCallback onRewindPast;
  final ValueListenable<double> dismissProgress; // shell, for scale
  const StoryGroupPage({...});
}

class _StoryGroupPageState extends ConsumerState<StoryGroupPage> with SingleTickerProviderStateMixin {
  late final StoryPlaybackController _playback;
  final Map<int, StoryMediaController> _media = {};
  double _zoom = 1;

  @override
  void initState() {
    super.initState();
    _playback = StoryPlaybackController(vsync: this, group: widget.group, onGroupComplete: widget.onComplete, onGroupRewindPast: widget.onRewindPast)
      ..addListener(_onPlaybackChanged);
    widget.activeIndex.addListener(_onActiveChanged);
    _playback.start(initialIndex: widget.initialItem);
    _onActiveChanged();
  }

  bool get _isActive => widget.activeIndex.value == widget.myIndex;

  void _onActiveChanged() {
    _playback.setActive(_isActive);
    if (_isActive) {
      _ensureMedia(_playback.index);
      _media[_playback.index]?.play();
      ref.read(storyGatewayProvider).markViewed(_playback.item.id); // fire-and-forget, deduped by id inside the gateway
    } else {
      for (final m in _media.values) { m.pause(); }
    }
  }

  void _onPlaybackChanged() {
    // item index changed → swap media, prefetch neighbours, mark viewed
    final i = _playback.index;
    _ensureMedia(i); _ensureMedia(i + 1); _ensureMedia(i - 1);
    for (final e in _media.entries.toList()) {
      if ((e.key - i).abs() > 1) { e.value.dispose(); _media.remove(e.key); }
    }
    if (_isActive) { _media[i]?.play(); ref.read(storyGatewayProvider).markViewed(_playback.item.id); }
    setState(() {}); // page-local rebuild only (caption/header/media swap). Progress bar listens separately.
  }

  void _ensureMedia(int i) {
    if (i < 0 || i >= widget.group.stories.length || _media.containsKey(i)) return;
    final c = StoryMediaController(widget.group.stories[i]);
    _media[i] = c;
    c.prepare(context).then((_) {
      if (!mounted) return;
      if (i == _playback.index && c.isVideo && c.video != null) {
        _playback.setDuration(c.video!.value.duration);
        if (_isActive) c.play();
      }
    });
  }

  @override
  void dispose() {
    widget.activeIndex.removeListener(_onActiveChanged);
    _playback.dispose();
    for (final m in _media.values) { m.dispose(); }
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final item = _playback.item;
    final media = _media[_playback.index];
    final colors = ref.watch(themeProvider).colors;
    return ValueListenableBuilder<double>(
      valueListenable: widget.dismissProgress,
      builder: (context, d, child) => Transform.scale(scale: 1 - 0.15 * d, child: child),
      child: Stack(fit: StackFit.expand, children: [
        // Media
        RepaintBoundary(child: Center(child: Transform.scale(
          scale: _zoom,
          child: media == null || !media.ready
              ? (media?.error != null ? _ErrorState(message: media!.error!, onRetry: () => setState(() { _media.remove(_playback.index); _ensureMedia(_playback.index); }))
                                       : const _LoadingState())
              : media.isVideo
                  ? AspectRatio(aspectRatio: media.video!.value.aspectRatio, child: VideoPlayer(media.video!))
                  : CachedNetworkImage(imageUrl: item.mediaUrl, fit: BoxFit.contain),
        ))),
        // Top gradient + progress + identity
        Positioned(top: 0, left: 0, right: 0, child: SafeArea(bottom: false, child: Column(children: [
          StoryProgressBar(count: widget.group.stories.length, index: _playback.index, progress: _playback.progress, boosted: item.boosted),
          _Header(group: widget.group, item: item, onClose: () => Navigator.of(context).maybePop()),
        ]))),
        // Caption
        if (item.caption?.isNotEmpty ?? false)
          Positioned(left: AzSpace.xl, right: AzSpace.xl, bottom: 140, child: Text(item.caption!, style: AzText.bodyL.copyWith(color: Colors.white, shadows: const [Shadow(blurRadius: 8, color: Colors.black54)]), maxLines: 4, overflow: TextOverflow.ellipsis)),
        // Business tray (only when linkedBizId and capability resolve)
        if (item.linkedBizId != null) Positioned(left: AzSpace.xl, right: AzSpace.xl, bottom: 88, child: StoryBusinessTray(bizId: item.linkedBizId!, onPause: () => _playback.hold(PauseReason.sheet), onResume: () => _playback.release(PauseReason.sheet))),
        // Interaction bar
        Positioned(left: 0, right: 0, bottom: 0, child: StoryInteractionBar(story: item, authorId: widget.group.authorId, onFocusChanged: (f) => f ? _playback.hold(PauseReason.sheet) : _playback.release(PauseReason.sheet))),
      ]),
    );
  }

  // Called by the shell's arbiter callbacks (via GlobalKey or a controller registry):
  void tap(bool right) => right ? _playback.next() : _playback.previous();
  void holdStart() => _playback.hold(PauseReason.hold);
  void holdEnd() => _playback.release(PauseReason.hold);
  void pinch(double s) { if (_media[_playback.index]?.isVideo ?? true) return; setState(() => _zoom = s.clamp(1.0, 3.0)); _playback.hold(PauseReason.zoom); }
  void pinchEnd() { setState(() => _zoom = 1); _playback.release(PauseReason.zoom); }
}
```

The shell forwards arbiter callbacks to the **active** page through a `Map<int, GlobalKey<_StoryGroupPageState>>`. Zoom reset uses `TweenAnimationBuilder` around the `Transform.scale` with `AzMotion.duration(context, MotionTokens.standard)` for the spring-back.

### 4.1 `StoryProgressBar`

```dart
class StoryProgressBar extends StatelessWidget {
  final int count; final int index; final Animation<double> progress; final bool boosted;
  const StoryProgressBar({...});
  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.fromLTRB(AzSpace.sm, AzSpace.sm, AzSpace.sm, 0),
    child: RepaintBoundary(child: AnimatedBuilder(
      animation: progress,
      builder: (_, __) => Row(children: [
        for (var i = 0; i < count; i++) ...[
          Expanded(child: ClipRRect(borderRadius: BorderRadius.circular(2), child: LinearProgressIndicator(
            minHeight: 2.5,
            value: i < index ? 1 : (i == index ? progress.value : 0),
            backgroundColor: Colors.white24,
            valueColor: AlwaysStoppedAnimation(boosted ? Colors.amberAccent : Colors.white),
          ))),
          if (i < count - 1) const SizedBox(width: 3),
        ],
      ]),
    )),
  );
}
```

---

## 5. `_ViewerShell` (vertical gestures, creator paging, dismiss)

```dart
class _ViewerShellState extends State<_ViewerShell> with SingleTickerProviderStateMixin {
  late final PageController _pages = PageController(initialPage: widget.initialGroupIndex);
  final ValueNotifier<int> _active = ValueNotifier(0);
  final ValueNotifier<double> _dismiss = ValueNotifier(0); // 0..1
  final Map<int, GlobalKey<_StoryGroupPageState>> _keys = {};
  late final AnimationController _dismissAnim = AnimationController(vsync: this, duration: MotionTokens.standard);

  @override
  void initState() {
    super.initState();
    _active.value = widget.initialGroupIndex;
    _pages.addListener(() {
      final p = _pages.page; if (p == null) return;
      final rounded = p.round();
      if (rounded != _active.value && (p - rounded).abs() < 0.5) _active.value = rounded; // active flips at the midpoint
    });
    _dismissAnim.addListener(() => _dismiss.value = _dismissAnim.value);
  }

  void _goToGroup(int i) {
    if (i < 0 || i >= widget.groups.length) { Navigator.of(context).maybePop(); return; }
    final travel = AzMotion.of(context).travel;
    travel ? _pages.animateToPage(i, duration: MotionTokens.spatial, curve: MotionTokens.enter) : _pages.jumpToPage(i);
  }

  void _verticalUpdate(double dy) {
    final h = MediaQuery.sizeOf(context).height;
    if (dy > 0 || _dismiss.value > 0) {
      _dismiss.value = (_dismiss.value + dy / (h * 0.5)).clamp(0.0, 1.0);
    } else if (dy < -12 && _dismiss.value == 0) {
      _openDetails();
    }
  }

  void _verticalEnd(double vy) {
    if (_dismiss.value == 0) return;
    final commit = _dismiss.value >= 0.35 || vy > 900;
    if (commit) { Navigator.of(context).maybePop(); return; }
    final travel = AzMotion.of(context).travel;
    if (!travel) { _dismiss.value = 0; return; }
    _dismissAnim.value = _dismiss.value;
    _dismissAnim.animateTo(0, curve: MotionTokens.exit);
  }

  void _openDetails() {
    final page = _keys[_active.value]?.currentState; if (page == null) return;
    final gw = ref.read(storyGatewayProvider); // via Consumer in build
    if (!gw.capabilities.contains(StoryCapability.react) && !gw.capabilities.contains(StoryCapability.viewers)) return;
    page._playback.hold(PauseReason.sheet);
    AzamanSheet.showPanel<void>(context, builder: (ctx, sc) => StoryDetailsSheet(story: page._playback.item, authorId: page.widget.group.authorId, scrollController: sc))
        .whenComplete(() => page._playback.release(PauseReason.sheet));
  }

  @override
  Widget build(BuildContext context) {
    return ValueListenableBuilder<double>(
      valueListenable: _dismiss,
      builder: (context, d, child) => Container(color: Colors.black.withValues(alpha: 1 - 0.6 * d), child: child),
      child: StoryGestureArbiter(
        pageController: _pages,
        callbacks: StoryGestureCallbacks(
          onTap: (right) => _keys[_active.value]?.currentState?.tap(right),
          onHoldStart: () => _keys[_active.value]?.currentState?.holdStart(),
          onHoldEnd: () => _keys[_active.value]?.currentState?.holdEnd(),
          onVerticalUpdate: _verticalUpdate,
          onVerticalEnd: _verticalEnd,
          onPinchScale: (s) => _keys[_active.value]?.currentState?.pinch(s),
          onPinchEnd: () => _keys[_active.value]?.currentState?.pinchEnd(),
        ),
        child: PageView.builder(
          controller: _pages,
          physics: const NeverScrollableScrollPhysics(),
          itemCount: widget.groups.length,
          itemBuilder: (context, i) {
            final key = _keys.putIfAbsent(i, GlobalKey.new);
            return StoryGroupPage(
              key: key, group: widget.groups[i], myIndex: i, activeIndex: _active, dismissProgress: _dismiss,
              initialItem: _firstUnseen(widget.groups[i]),
              onComplete: () => _goToGroup(i + 1),
              onRewindPast: () => _goToGroup(i - 1),
            );
          },
        ),
      ),
    );
  }

  int _firstUnseen(StoryGroup g) { final i = g.stories.indexWhere((s) => !s.seen); return i < 0 ? 0 : i; }
}
```

Route: `PageRouteBuilder(opaque: false, barrierColor: Colors.black, transitionsBuilder: fade MotionTokens.standard)`; reduced motion → `Duration.zero` via `AzMotion.duration`. The ring→header Hero (`AzIdentityTag.story(authorId)`) flies only when `travel`.

---

## 6. Interaction layer and `StoryGateway`

### 6.1 `lib/experience/gateways/story_gateway.dart`

```dart
enum StoryCapability { view, reply, react, boost, share, viewers, edit, privacy, lifespan, viewOnce }

class StoryReaction { final String storyId; final String emoji; final String clientEventId; const StoryReaction({...}); }

abstract interface class StoryGateway {
  Set<StoryCapability> get capabilities;
  Future<void> markViewed(String storyId);                       // deduped
  Future<AzGatewayResult<void>> reply(String storyId, String message);
  Future<AzGatewayResult<void>> react(StoryReaction reaction);   // unsupported today
  Future<AzGatewayResult<void>> boost(String storyId, int amount);
  Future<AzGatewayResult<Uri>> shareLink(String storyId);        // unsupported today
}

class HttpStoryGateway implements StoryGateway {
  HttpStoryGateway(this.ref);
  final Ref ref;
  final Set<String> _viewed = {};

  @override
  Set<StoryCapability> get capabilities => const {StoryCapability.view, StoryCapability.reply, StoryCapability.boost};

  @override
  Future<void> markViewed(String id) async {
    if (!_viewed.add(id)) return;
    await ref.read(storyFeedProvider.notifier).markViewed(id); // existing: POST /stories/:id/view, swallows errors
  }

  @override
  Future<AzGatewayResult<void>> reply(String id, String message) async {
    final ok = await ref.read(storyFeedProvider.notifier).replyStory(id, message);
    return ok ? const AzOk(null) : const AzFailed('Reply not sent');
  }

  @override
  Future<AzGatewayResult<void>> react(StoryReaction r) async => const AzUnsupported('Story reactions need POST /stories/:id/react with {emoji, clientEventId}');

  @override
  Future<AzGatewayResult<void>> boost(String id, int amount) async {
    try { await ref.read(storyFeedProvider.notifier).boost(id, amount); return const AzOk(null); }
    catch (e) { return AzFailed('Boost failed', cause: e); }
  }

  @override
  Future<AzGatewayResult<Uri>> shareLink(String id) async => const AzUnsupported('No share endpoint');
}

final storyGatewayProvider = Provider<StoryGateway>((ref) => HttpStoryGateway(ref));
```

Reaction authenticity (brief §9.4/§23): when the backend adds reactions, the request carries `clientEventId` (uuid) and the server echoes `eventId`; the client applies the server echo through `RealtimeEventDeduper.accept(eventId)` and never increments a local counter on tap alone. The sticker fires a 1-shot scale animation as **feedback**, not as a count.

### 6.2 `StoryInteractionBar`

Reply field (existing behaviour: `replyStory`, snackbar outcome) + optional reaction stickers (`❤️ 🔥 😂 👏 😮 😢`) rendered **only** if `capabilities.contains(react)` + share **only** if `share`. On focus → `onFocusChanged(true)` pauses playback. Field is `TextField` with `AzText.body`, white on `Colors.white12` pill, `AzRadius.pill`.

### 6.3 `StoryDetailsSheet`

For the author's own story: viewers list (only if `viewers` supported — unsupported today → the sheet is not offered, see `_openDetails`). For others: large reaction grid. Uses `AzamanSheet.showPanel`.

---

## 7. `StoryBusinessTray` — story → store continuity (brief §9.5, §11)

```dart
final linkedBusinessProvider = FutureProvider.autoDispose.family<BusinessProfile?, String>((ref, bizId) async {
  final cached = ref.read(businessSearchProvider).results.where((b) => b.bizId == bizId || b.id == bizId).firstOrNull;
  if (cached != null) return cached;
  return BusinessService().getBusinessByBizId(bizId); // adapt to the service's singleton/provider access
});

class StoryBusinessTray extends ConsumerWidget {
  final String bizId; final VoidCallback onPause; final VoidCallback onResume;
  const StoryBusinessTray({...});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final biz = ref.watch(linkedBusinessProvider(bizId)).valueOrNull;
    if (biz == null) return const SizedBox.shrink();
    final profile = MarketplaceExperienceCatalog.fromCategory(biz.category);
    final travel = AzMotion.of(context).travel;
    return Semantics(button: true, label: '${profile.primaryActionLabel} at ${biz.businessName}', child: ScaleTap(
      onTap: () {
        AzamanHaptics.selection();
        onPause();
        // Identity-preserving: logo Hero from the tray to StoreIdentityRow (03 §1.1)
        context.push(AzRoutes.business(biz.bizId)).whenComplete(onResume); // verify builder in `AzRoutes` (lib/router/route_registry.dart)
      },
      child: Container(
        padding: const EdgeInsets.symmetric(horizontal: AzSpace.md, vertical: AzSpace.sm),
        decoration: BoxDecoration(color: Colors.white.withValues(alpha: .14), borderRadius: BorderRadius.circular(AzRadius.pill)),
        child: Row(mainAxisSize: MainAxisSize.min, children: [
          AzIdentityMorph(tag: AzIdentityTag.business(biz.bizId), travel: travel, child: ChatAvatar(imageUrl: biz.logoUrl, name: biz.businessName, size: 28)),
          const SizedBox(width: AzSpace.sm),
          Flexible(child: Text(biz.businessName, maxLines: 1, overflow: TextOverflow.ellipsis, style: AzText.label.copyWith(color: Colors.white))),
          const SizedBox(width: AzSpace.sm),
          Text(profile.primaryActionLabel, style: AzText.label.copyWith(color: Colors.amberAccent)),
          TrustMark(business: biz),
        ]),
      ),
    ));
  }
}
```

Shows one vertical-appropriate action (Shop / View menu / See rooms / Book a seat) from the **existing** catalog — nothing invented. "Send money" CTA from a story: not added until the backend exposes a story payment-intent object; a `// TODO(story-pay-intent)` seam in the tray documents the gate (never auto-execute a financial action from a story).

---

## 8. Tests (S1)

`test/widgets/stories/viewer/`:
- `story_playback_controller_test.dart`: progress completes → index+1; last item → `onGroupComplete`; `hold` + `release` with multiple reasons; `setDuration` while running keeps value; `setActive(false)` stops.
- `story_gesture_arbiter_test.dart` (pure pointer tests with `TestGesture`): tap right/left zones; hold ≥ 350 ms → holdStart/holdEnd; horizontal 60px → page drag updates; vertical → vertical callbacks; second pointer → pinch, no page flip; cancel paths.
- `story_viewer_lifecycle_test.dart`: with 3 groups, swipe to group 2 → group 1's `VideoPlayerController` is paused (mock via `VideoPlayerPlatform` fake) and disposed when 2 away; back restores; `markViewed` called once per story id.
- `story_dismiss_test.dart`: drag down 40% → pops; 20% → settles back; reduced motion settles in one pump.
- `story_business_tray_test.dart`: tray hidden when business unresolved; label matches catalog primary action.
- Goldens (brief §26 #15–17): single segment, multi-segment at 50% progress (set `_progress.value`), mid horizontal drag (`PageController.jumpTo(page.offset + 120)`).

---

## 9. Acceptance

- Opening from the inbox: ring Hero lands on the viewer header; first unseen item plays.
- Tap zones, hold, horizontal swipe between creators, swipe down to dismiss, pinch zoom on images — each gesture behaves identically every time; no "wrong gesture won" cases on a 10-minute soak.
- Video audio never plays from a non-visible page; memory stays flat while swiping through 20 creators.
- Reduced motion: no page slide, no Hero, no dismiss scale; everything still works.
- Business stories show a single tray with the right verb and open the storefront with the logo continuing.