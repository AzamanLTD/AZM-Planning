# 06 — Story Creation: Camera, Media Selector, Scene Editor Model, Edit Session, Privacy, Lifespan, View-Once, Analytics

Implements brief §10. Depends on 01 (gateways, `DemoGuard`) and 05 (`StoryGateway`, `StoryCapability`).

Existing chain (from `FriendsHubScreen._pickAndCreateStory`): `StoryCameraScreen({onCaptured})` (493 lines, `image_picker` only, simulated preview, `StoryFilter` enum with `colorFilter`) → `StoryEditorScreen` (631 lines: `EditMode{none,text,sticker,draw}`, ad-hoc `_OverlayItem{text?, sticker?, position, color…}`, one stroke `List<Offset> _drawPoints`, `_DoodlePainter`) → `StoryCreationScreen` (181 lines: caption + multipart POST `/stories` with fields `caption`, optional `linkedBizId`, invalidates `storyFeedProvider`). Routes: `/story-camera`, `/story-editor`, `/story-create`, `/story-highlights`, `/story-analytics/:businessId`.

Strategy: keep the three-screen chain and the upload call; replace the editor's internals with a **document model** (`StoryScene`) and a reducer; make camera/privacy/lifespan/view-once **capability-gated** so nothing claims what the platform cannot do.

New files: `lib/stories/scene/story_scene.dart`, `story_scene_reducer.dart`, `story_scene_renderer.dart`, `story_scene_codec.dart`; `lib/stories/story_draft.dart`, `story_privacy.dart`, `story_edit_session.dart`, `story_capture_capabilities.dart`; `lib/widgets/stories/editor/*` (tool rail, text tool, draw tool, sticker tray, filter rail, overlay layer); `lib/widgets/stories/create/*` (privacy sheet, lifespan picker, media selector).

---

## 1. Camera — decision point, not a blind package add

`pubspec.yaml` has no `camera` package. `StoryCameraScreen` delegates capture to `image_picker` (`ImageSource.camera`, `preferredCameraDevice`) and paints a fake preview with grid/flash/flip placeholders that do nothing.

### 1.1 Capability model — `lib/stories/story_capture_capabilities.dart`

```dart
enum CaptureCapability { photo, video, tapHold, zoom, switchCamera, flash, screenIlluminate, dualCamera, liveFilters }

class StoryCaptureCapabilities {
  final Set<CaptureCapability> supported;
  const StoryCaptureCapabilities(this.supported);
  bool has(CaptureCapability c) => supported.contains(c);

  /// What image_picker gives us today. Live filters are applied post-capture only.
  static const pickerOnly = StoryCaptureCapabilities({CaptureCapability.photo, CaptureCapability.video, CaptureCapability.switchCamera});

  /// What the `camera` plugin would give us (reference for the decision below).
  static const livePreview = StoryCaptureCapabilities({
    CaptureCapability.photo, CaptureCapability.video, CaptureCapability.tapHold, CaptureCapability.zoom,
    CaptureCapability.switchCamera, CaptureCapability.flash, CaptureCapability.screenIlluminate, CaptureCapability.liveFilters,
  });
}

final storyCaptureCapabilitiesProvider = Provider<StoryCaptureCapabilities>((_) => StoryCaptureCapabilities.pickerOnly);
```

### 1.2 The decision the agent must make (and report)

**Option A (recommended for this pass):** keep `image_picker`. Remove the fake preview, flash toggle, grid and flip placeholders from `StoryCameraScreen`; the screen becomes an honest **capture chooser**: big "Take photo" / "Record video" / "Choose from gallery" with the media selector (§2). Filters move entirely into the editor (they are `ColorFilter`s — they never needed live preview). No dead controls remain.

**Option B:** add `camera: ^0.11.x` (check Android `minSdk` ≥ 21, iOS `NSCameraUsageDescription` already present for image_picker, `NSMicrophoneUsageDescription` for video). Then `StoryCaptureCapabilities.livePreview` applies and the controls become real:
- tap = photo (`controller.takePicture()`), hold ≥ 400 ms = `startVideoRecording()`, release = `stopVideoRecording()`;
- one-handed zoom = vertical drag while recording → `setZoomLevel(lerp(min,max, t))`;
- switch = `CameraController(cameras[next])` swap;
- flash = `setFlashMode`; front "screen illumination" = overlay `Container(color: Colors.white.withValues(alpha: .85))` while capturing (only meaningful on front camera);
- dual-camera/PiP: **not** supported by the plugin → `dualCamera` stays unsupported; do not fake PiP.

Whichever option: UI reads `storyCaptureCapabilitiesProvider` and renders only supported controls. The `StoryFilter` enum moves from the camera file to `lib/stories/story_filter.dart` (same values, same `colorFilter`), imported by both.

### 1.3 Camera → editor handoff

`onCaptured(StoryDraft draft)` replaces the current `(File, bool isVideo, StoryFilter)` tuple (verify the current signature at `story_camera_screen.dart:126` and the call in `_pickAndCreateStory`).

---

## 2. Media selector — `lib/widgets/stories/create/story_media_selector.dart`

```dart
enum StoryMediaKind { image, video }

class StoryMediaSource {
  final File file;
  final StoryMediaKind kind;
  final int? durationMs;      // video, from VideoPlayerController after init
  final Size? pixelSize;      // image, from decodeImageFromList
  const StoryMediaSource({required this.file, required this.kind, this.durationMs, this.pixelSize});
}
```

- Gallery: `ImagePicker.pickMedia()` (one item); photo/video distinction from `XFile.mimeType` or extension.
- Crop/rotate: `image_cropper` (already a dependency) with `aspectRatioPresets: [original, ratio9x16]` and locked 9:16 by default — "story-safe aspect ratio handling". Cropping writes a new `File`; the original is not mutated.
- Recent media: `pickMultipleMedia(limit: 12)` is **not** a gallery browser; the brief's "recent media" strip is implemented only if `photo_manager` is added (decision like §1.2). Default: omit the strip; the gallery button opens the system picker.
- Preview: `AspectRatio(9/16)` with `BoxFit.contain` on a black surface before entering the editor.

---

## 3. Scene document — `lib/stories/scene/story_scene.dart`

A persistable, testable, re-renderable description of what the user composed. Coordinates are **normalised** (0..1 of the 9:16 canvas) so the scene renders identically at any size and survives device changes.

```dart
import 'dart:ui';

/// Normalised point on the 9:16 canvas.
class NPoint { final double x, y; const NPoint(this.x, this.y); }

sealed class StoryLayer {
  final String id;
  final int z;
  const StoryLayer({required this.id, required this.z});
}

class TextLayer extends StoryLayer {
  final String text; final NPoint position; final double scale; final double rotation; final int colorArgb; final String? fontFamily; final bool background;
  const TextLayer({required super.id, required super.z, required this.text, required this.position, this.scale = 1, this.rotation = 0, required this.colorArgb, this.fontFamily, this.background = false});
}

class StickerLayer extends StoryLayer {
  final String emoji; final NPoint position; final double scale; final double rotation;
  const StickerLayer({required super.id, required super.z, required this.emoji, required this.position, this.scale = 1, this.rotation = 0});
}

class LinkLayer extends StoryLayer {
  final Uri url; final String label; final NPoint position;
  const LinkLayer({required super.id, required super.z, required this.url, required this.label, required this.position});
}

enum BrushMode { pen, marker, neon, eraser }

class Stroke {
  final List<NPoint> points; final int colorArgb; final double widthN; final BrushMode mode;
  const Stroke({required this.points, required this.colorArgb, required this.widthN, required this.mode});
}

class DrawLayer extends StoryLayer {
  final List<Stroke> strokes;
  const DrawLayer({required super.id, required super.z, required this.strokes});
}

class BlurLayer extends StoryLayer {
  final Rect normRect; final double sigma;
  const BlurLayer({required super.id, required super.z, required this.normRect, this.sigma = 12});
}

/// Widgets that need backend data to resolve (poll, location, weather) are
/// modelled as *intent* layers; they render only when the matching
/// StoryCapability is reported by the gateway.
class WidgetIntentLayer extends StoryLayer {
  final String kind;                 // 'poll' | 'location' | 'weather'
  final Map<String, Object?> params; // typed per kind in the renderer
  final NPoint position;
  const WidgetIntentLayer({required super.id, required super.z, required this.kind, required this.params, required this.position});
}

class MediaTransform {
  final double scale; final NPoint offset; final int quarterTurns; final Rect? cropNorm;
  const MediaTransform({this.scale = 1, this.offset = const NPoint(0, 0), this.quarterTurns = 0, this.cropNorm});
}

class AudioMix {
  final String? trackId; final double originalVolume; final double trackVolume;
  const AudioMix({this.trackId, this.originalVolume = 1, this.trackVolume = 0});
}

class StoryScene {
  final String filterId;             // StoryFilter.name
  final MediaTransform media;
  final List<StoryLayer> layers;     // sorted by z
  final AudioMix audio;
  final String? caption;
  const StoryScene({this.filterId = 'none', this.media = const MediaTransform(), this.layers = const [], this.audio = const AudioMix(), this.caption});

  StoryScene copyWith({...}) => ...;
  int get nextZ => layers.isEmpty ? 0 : layers.map((l) => l.z).reduce((a, b) => a > b ? a : b) + 1;
}
```

### 3.1 Reducer — `story_scene_reducer.dart` (pure, undo-friendly)

```dart
sealed class SceneAction { const SceneAction(); }
class AddLayer extends SceneAction { final StoryLayer layer; const AddLayer(this.layer); }
class RemoveLayer extends SceneAction { final String id; const RemoveLayer(this.id); }
class MoveLayer extends SceneAction { final String id; final NPoint position; final double scale; final double rotation; const MoveLayer(...); }
class UpdateText extends SceneAction { final String id; final String text; final int colorArgb; const UpdateText(...); }
class AppendStroke extends SceneAction { final Stroke stroke; const AppendStroke(this.stroke); }
class SetFilter extends SceneAction { final String filterId; const SetFilter(this.filterId); }
class SetMediaTransform extends SceneAction { final MediaTransform t; const SetMediaTransform(this.t); }
class SetCaption extends SceneAction { final String? caption; const SetCaption(this.caption); }
class SetAudio extends SceneAction { final AudioMix mix; const SetAudio(this.mix); }

StoryScene reduceScene(StoryScene s, SceneAction a) => switch (a) {
      AddLayer(:final layer) => s.copyWith(layers: [...s.layers, layer]..sort((x, y) => x.z.compareTo(y.z))),
      RemoveLayer(:final id) => s.copyWith(layers: s.layers.where((l) => l.id != id).toList()),
      MoveLayer(:final id, :final position, :final scale, :final rotation) => s.copyWith(layers: s.layers.map((l) => l.id == id ? _moved(l, position, scale, rotation) : l).toList()),
      UpdateText(:final id, :final text, :final colorArgb) => s.copyWith(layers: s.layers.map((l) => l is TextLayer && l.id == id ? TextLayer(id: l.id, z: l.z, text: text, position: l.position, scale: l.scale, rotation: l.rotation, colorArgb: colorArgb, fontFamily: l.fontFamily, background: l.background) : l).toList()),
      AppendStroke(:final stroke) => _appendStroke(s, stroke),
      SetFilter(:final filterId) => s.copyWith(filterId: filterId),
      SetMediaTransform(:final t) => s.copyWith(media: t),
      SetCaption(:final caption) => s.copyWith(caption: caption),
      SetAudio(:final mix) => s.copyWith(audio: mix),
    };

StoryScene _appendStroke(StoryScene s, Stroke st) {
  final draw = s.layers.whereType<DrawLayer>().firstOrNull;
  if (draw == null) return reduceScene(s, AddLayer(DrawLayer(id: 'draw', z: s.nextZ, strokes: [st])));
  return s.copyWith(layers: s.layers.map((l) => l.id == draw.id ? DrawLayer(id: l.id, z: l.z, strokes: [...draw.strokes, st]) : l).toList());
}
```

Eraser: a `Stroke` with `BrushMode.eraser` is rendered with `BlendMode.clear` inside a `saveLayer` — so erase is itself an appended stroke and undo works uniformly.

### 3.2 Editor state (history) — `lib/providers/story_editor_provider.dart`

```dart
class StoryEditorState {
  final StoryScene scene; final List<StoryScene> past; final List<StoryScene> future; final String? selectedLayerId; final EditTool tool;
  const StoryEditorState({required this.scene, this.past = const [], this.future = const [], this.selectedLayerId, this.tool = EditTool.none});
  bool get canUndo => past.isNotEmpty; bool get canRedo => future.isNotEmpty;
}
enum EditTool { none, text, sticker, draw, filter, crop, link, blur, audio, widget }

class StoryEditorNotifier extends StateNotifier<StoryEditorState> {
  StoryEditorNotifier(StoryScene initial) : super(StoryEditorState(scene: initial));
  static const _maxHistory = 50;
  void apply(SceneAction a, {bool record = true}) {
    final next = reduceScene(state.scene, a);
    state = StoryEditorState(
      scene: next,
      past: record ? [...state.past, state.scene].take(_maxHistory).toList() : state.past,
      future: record ? const [] : state.future,
      selectedLayerId: state.selectedLayerId, tool: state.tool,
    );
  }
  void undo() { if (!state.canUndo) return; final prev = state.past.last; state = StoryEditorState(scene: prev, past: state.past.sublist(0, state.past.length - 1), future: [state.scene, ...state.future], tool: state.tool); }
  void redo() { if (!state.canRedo) return; final nxt = state.future.first; state = StoryEditorState(scene: nxt, past: [...state.past, state.scene], future: state.future.sublist(1), tool: state.tool); }
  void select(String? id) => state = StoryEditorState(scene: state.scene, past: state.past, future: state.future, selectedLayerId: id, tool: state.tool);
  void setTool(EditTool t) => state = StoryEditorState(scene: state.scene, past: state.past, future: state.future, selectedLayerId: state.selectedLayerId, tool: t);
}

final storyEditorProvider = StateNotifierProvider.autoDispose.family<StoryEditorNotifier, StoryEditorState, StoryScene>((ref, initial) => StoryEditorNotifier(initial));
```

Drag of a layer applies `MoveLayer` with `record: false` on every update and a final `record: true` on end, so one gesture = one undo step.

### 3.3 Renderer — `story_scene_renderer.dart`

Single `StorySceneRenderer(scene, media, size, {interactive})` that both the editor (interactive: true, handles on selected layer) and the viewer/export (false) use. Layers map to widgets positioned at `NPoint × size`; `DrawLayer` is one `CustomPaint` (`_StrokesPainter`, `shouldRepaint` on strokes identity) in a `RepaintBoundary`; `BlurLayer` is `ClipRect` + `BackdropFilter` only within its rect; filter is `ColorFiltered(colorFilter: StoryFilter.byId(filterId).colorFilter)` over the media.

Export: `RenderRepaintBoundary.toImage(pixelRatio: 3)` of the non-interactive renderer at 1080×1920 → PNG → the existing multipart upload replaces the raw media when the scene has layers or a filter; video stays as-is with the scene JSON uploaded as `sceneJson` (overlay rendering on video is server/future work — declare it in the PR report).

### 3.4 Codec — `story_scene_codec.dart`

`toJson`/`fromJson` with a `schemaVersion: 1`. Round-trip test for every layer type. This JSON is what `StoryEditSession` (§5) sends and what the viewer could re-render later.

---

## 4. Editor UI — `lib/widgets/stories/editor/`

`StoryEditorScreen` keeps its name/route; its body becomes:

```
Stack
├── StorySceneRenderer(interactive: true)          canvas (AspectRatio 9/16, letterboxed)
├── EditorTopBar      back · undo · redo · filter · more(crop/rotate, blur, audio, link, widgets*)
├── ToolRail (right)  text · sticker · draw · *gated*
├── ToolPanel (bottom, per tool)
│     text: TextToolPanel (TextField, colour row, background toggle)
│     draw: DrawToolPanel (BrushMode segmented, width slider, colour row)
│     sticker: StickerTray (emoji grid via dart_emoji, recent 12 persisted)
│     filter: FilterRail (thumbnails from the same media, StoryFilter.values)
│     crop: CropPanel (image_cropper launch; result → SetMediaTransform)
└── NextButton → StoryCreationScreen(draft)
```

- Layer gestures: `GestureDetector` with `onScaleStart/Update/End` on each layer (translate + scale + rotate in one recognizer — no competing pan + scale). Drawing uses a full-canvas `Listener` only while `tool == draw`; otherwise it is absent from the tree so taps select layers.
- Deletion: drag a layer onto the bottom trash zone (haptic `AzamanHaptics.threshold` on enter) or select + trash button.
- Gating: `link`, `widget` (poll/location/weather), `audio` tools appear only if `storyGatewayProvider.capabilities` contains the matching `StoryCapability` (`link`, `widgets`, `audio` — add these to the enum). Today none are supported → the tool rail shows text/sticker/draw/filter/crop/blur only.
- Blur/redaction: drag-draw a rectangle → `BlurLayer`. Local-only, always available.

---

## 5. `StoryDraft`, `StoryEditSession` and upload

### 5.1 `lib/stories/story_draft.dart`

```dart
class StoryDraft {
  final StoryMediaSource media;
  final StoryScene scene;
  final StoryPrivacy privacy;
  final StoryLifespan lifespan;
  final bool viewOnce;
  final String? linkedBizId;
  const StoryDraft({required this.media, this.scene = const StoryScene(), this.privacy = const StoryPrivacy.everyone(), this.lifespan = StoryLifespan.day, this.viewOnce = false, this.linkedBizId});
  StoryDraft copyWith({...}) => ...;
}
```

### 5.2 Upload — surgical change in `StoryCreationScreen._uploadStory` (anchor ~L32–L63)

Keep the `MultipartRequest` to `${ApiClient.baseUrl}/stories` via `apiClient.multipart('/stories', request)`. Fields become:

```dart
request.fields['caption'] = draft.scene.caption ?? '';
if (draft.linkedBizId != null) request.fields['linkedBizId'] = draft.linkedBizId!;
request.fields['sceneJson'] = jsonEncode(StorySceneCodec.toJson(draft.scene));   // server may ignore today
if (gw.capabilities.contains(StoryCapability.privacy)) request.fields['privacy'] = jsonEncode(draft.privacy.toJson());
if (gw.capabilities.contains(StoryCapability.lifespan)) request.fields['expiresInSeconds'] = draft.lifespan.duration.inSeconds.toString();
if (gw.capabilities.contains(StoryCapability.viewOnce)) request.fields['viewOnce'] = draft.viewOnce.toString();
```

The media part is the exported PNG when `scene` has layers/filter and media is an image; otherwise the original file. Keep `ref.invalidate(storyFeedProvider)` on success.

### 5.3 `lib/stories/story_edit_session.dart` — editing after posting (brief §10.4)

```dart
enum StoryEditField { caption, overlays, drawing, privacy, mediaMeta }

class StoryEditSession {
  final String storyId;
  final StoryScene original;
  final StoryPrivacy originalPrivacy;
  StoryScene scene;
  StoryPrivacy privacy;
  StoryEditSession({required this.storyId, required this.original, required this.originalPrivacy}) : scene = original, privacy = originalPrivacy;

  Set<StoryEditField> get dirty => {
        if (scene.caption != original.caption) StoryEditField.caption,
        if (!_sameLayers(scene.layers.where((l) => l is! DrawLayer), original.layers.where((l) => l is! DrawLayer))) StoryEditField.overlays,
        if (!_sameLayers(scene.layers.whereType<DrawLayer>(), original.layers.whereType<DrawLayer>())) StoryEditField.drawing,
        if (privacy != originalPrivacy) StoryEditField.privacy,
        if (scene.media != original.media || scene.filterId != original.filterId) StoryEditField.mediaMeta,
      };

  /// One commit, one PATCH. Never called from a widget other than the session's own Save.
  Future<AzGatewayResult<void>> commit(StoryGateway gw) {
    if (dirty.isEmpty) return Future.value(const AzOk(null));
    return gw.editStory(storyId, scene: dirty.contains(StoryEditField.privacy) && dirty.length == 1 ? null : scene, privacy: dirty.contains(StoryEditField.privacy) ? privacy : null);
  }
}
```

`StoryGateway.editStory` → `AzUnsupported('PATCH /stories/:id not available')` today; the viewer's own-story "Edit" affordance (05 §6.3) is hidden unless `StoryCapability.edit` is present.

---

## 6. Privacy — `lib/stories/story_privacy.dart`

```dart
enum StoryAudience { everyone, contacts, closeFriends, selected }

class StoryPrivacy {
  final StoryAudience audience;
  final Set<int> selectedUserIds;    // when audience == selected
  final Set<int> excludedUserIds;    // permanent exclusions, apply to all audiences
  const StoryPrivacy({required this.audience, this.selectedUserIds = const {}, this.excludedUserIds = const {}});
  const StoryPrivacy.everyone() : this(audience: StoryAudience.everyone);
  Map<String, Object?> toJson() => {'audience': audience.name.toUpperCase(), 'selected': selectedUserIds.toList(), 'excluded': excludedUserIds.toList()};
  factory StoryPrivacy.fromJson(Map<String, dynamic> j) => ...;
  @override bool operator ==(Object o) => o is StoryPrivacy && o.audience == audience && setEquals(o.selectedUserIds, selectedUserIds) && setEquals(o.excludedUserIds, excludedUserIds);
  @override int get hashCode => Object.hash(audience, selectedUserIds.length, excludedUserIds.length);
}
```

Defaults persist locally (`SharedPreferences` key `az_story_privacy_default`) so the next story reuses the last choice. `StoryPrivacySheet` (`AzamanSheet.showPanel`) lists the four audiences with a contacts picker (`friendProvider.friends` → typed `InboxEntry` ids) for *selected* and *excluded*. It is a separate sheet from any visual tool — never inside the editor rail. Shown in `StoryCreationScreen` only when `StoryCapability.privacy` is supported; otherwise a single line "Visible to your contacts" states the server's actual behaviour (confirm with backend; never guess the default).

---

## 7. Lifespan and view-once

```dart
enum StoryLifespan {
  hours6(Duration(hours: 6)), day(Duration(hours: 24)), days3(Duration(days: 3)), week(Duration(days: 7));
  const StoryLifespan(this.duration); final Duration duration;
  String get label => switch (this) { hours6 => '6 hours', day => '24 hours', days3 => '3 days', week => '1 week' };
}
```

- `LifespanPicker` segmented control, shown only if `StoryCapability.lifespan`. Default `day`. No UI anywhere hardcodes "24h": the viewer/analytics read `expiresAt` from the server item (add `expiresAt: DateTime?` to `StoryItem.fromJson` reading `j['expiresAt']`, nullable).
- View-once: `viewOnce` toggle only if `StoryCapability.viewOnce`. Copy: "Viewers can open this once." **Never** "screenshots are blocked" — screenshot prevention is not enforceable from Flutter without platform work; if the product wants it, a `CaptureCapability.secureSurface` flag gates a `FLAG_SECURE`/`UIScreen.isCaptured` platform channel in a later PR.

---

## 8. Analytics — `story_analytics_screen.dart` (GET `/stories/analytics/business/:id`)

Typed model replaces map reads:

```dart
class StoryAnalyticsItem { final String storyId; final DateTime createdAt; final DateTime? expiresAt; final int views; final int uniqueViewers; final int replies; final int? reactions; final String? thumbnailUrl; }
class StoryAnalytics { final int totalViews; final int uniqueViewers; final int replies; final int? reactions; final List<StoryAnalyticsItem> perStory; }
```

- `reactions` nullable → row omitted when null (backend has no reactions yet).
- Screen: summary tiles (only non-null metrics) + per-story list sorted by `createdAt desc` with "expires in …" from `expiresAt` (omitted when null) + a local filter field over captions/dates (client-side; search architecture lives in the notifier so a server `?q=` can replace it later).
- No sparkline unless the API returns a time series.

---

## 9. Tests (S2)

- `test/stories/scene/story_scene_reducer_test.dart`: add/remove/move/text/stroke/eraser/filter; z ordering; `_appendStroke` creates the draw layer once.
- `story_editor_provider_test.dart`: undo/redo, history cap, drag coalescing (record: false then true = one step).
- `story_scene_codec_test.dart`: round-trip all layer types; unknown `schemaVersion` → `FormatException`.
- `story_privacy_test.dart`: equality, json round trip, exclusions survive audience change.
- `story_edit_session_test.dart`: dirty set per field; commit with no changes does not call the gateway.
- Widget: editor tool rail hides gated tools when the gateway reports no capability; `StoryCreationScreen` omits lifespan/view-once controls likewise.
- Golden (brief §26 #18): editor with one text, one sticker, one stroke via `pumpGoldenSurface` (`Image.memory` fixture).

---

## 10. Acceptance

- The camera screen has no control that does nothing.
- Every edit is undoable; closing and reopening the editor on the same draft shows the same scene (scene survives as a value).
- Posting a story with overlays uploads a flattened image; the feed shows it as composed.
- Privacy, lifespan and view-once only appear when the server supports them; copy never over-claims.
- Analytics never shows a metric the API did not return.