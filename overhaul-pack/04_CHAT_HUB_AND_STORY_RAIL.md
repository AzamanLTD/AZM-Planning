# 04 — Chat Hub (Inbox) Redesign and the Collapsed-by-Default Pull-Down Story Rail

Implements brief §8 (chat hub, story rail behavior), the hub part of §14, §21 (guardrails). Depends on 01 (`AzSnapSolver`, `AzIdentityTag`, test conventions). Document 07 adds the Susu identity and the per-row action gateway this document leaves seams for.

Primary file: `lib/screens/friends/friends_hub_screen.dart` (`FriendsHubScreen`, 1766 lines in the snapshot). New files live in `lib/widgets/inbox/` and `lib/widgets/stories/`.

Unchanged: `friendProvider` (friends, requests, search), `groupListProvider`/`GroupSummary`, `storyFeedProvider`, `_openRequestsSheet`, `_showAddFriendDialog`, `_pickAndCreateStory`, friend search (`_buildSearchResults`). Only the **body layout**, the **story rail**, and the **row widgets** change.

---

## 1. The problem in the current body

Anchor (`friends_hub_screen.dart` `build`, ~L313–L669):

```dart
    return Scaffold(
      backgroundColor: Colors.transparent,
      body: SafeArea(
        bottom: false,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            const SizedBox(height: 8),
            Padding( /* header row: 'Inbox' 28/w800, contacts, requests, search */ ),
            if (!_isSearching) ...[
              const SizedBox(height: 12),
              Consumer( /* storyFeedProvider → SizedBox(height: 100, child: ListView.builder(horizontal)) */ ),
            ],
            if (_isSearching) ...[ /* TextField */ ],
            const SizedBox(height: 8),
            Expanded(
              child: _isSearching
                  ? _buildSearchResults(colors, provider)
                  : _buildChatsTab(colors, provider),
            ),
          ],
        ),
      ),
    );
```

- The rail is a fixed 100dp box outside the scroll owner: it can neither collapse nor be pulled.
- `_buildChatsTab` returns a `ListView.separated` — a second vertical scrollable would be needed to animate the rail, which is exactly the nested-scroll fight the brief forbids.
- Rows are built from raw `Map<String, dynamic>` (`friend['friend']`, `friend['latestMessage']`).

---

## 2. Design: the chat list is the only scroll owner; the rail lives at negative scroll offsets

Flutter's `CustomScrollView.center` lays out slivers **before** the center sliver in the reverse growth direction, which makes the viewport's `minScrollExtent` negative by exactly their extent. Put the full story rail in a sliver before the center, and the chat list (center) starts at offset `0` with the rail just above the top edge.

- Offset `0` → rail hidden ("closed"). This is the natural initial state: **no `initialScrollOffset`**, no hardcoded 115px, dynamic rail height just changes `minScrollExtent`.
- Dragging down at the top scrolls to negative offsets and reveals the rail progressively. Same gesture, same owner.
- Custom `ScrollPhysics` snap the resting position to `{minScrollExtent, 0}` using `AzSnapSolver` with velocity + threshold. Partial states are allowed mid-gesture and settle predictably.
- A compact strip (stacked avatars + "Stories") is the first child of the center sliver so the rail is discoverable while collapsed. Tapping it animates to open.
- The rail's own content is a horizontal `ListView` (different axis — allowed) wrapped in `RepaintBoundary`; its fade/scale read the `ScrollController` through an `AnimatedBuilder`, so only the rail repaints on drag ticks; chat rows never rebuild.
- Reduced motion: the ballistic snap is replaced by an immediate settle; state still becomes open/closed.

Keys: `inbox_scroll`, `inbox_story_rail`, `inbox_story_rail_compact`, `inbox_center`.

---

## 3. New widgets

### 3.1 `lib/widgets/stories/story_rail_strip.dart` — the expanded rail (shared)

Extracted from the current hub body (the `buildMyStatus` closure + the `ListView.builder` at ~L470–L590). Used by the hub now and available to any other surface instead of the duplicates in `messages_hub_screen.dart`.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:azaman/models/story_model.dart';
import 'package:azaman/providers/auth_provider.dart';
import 'package:azaman/providers/story_provider.dart';
import 'package:azaman/providers/theme_provider.dart';
import 'package:azaman/theme/az_space.dart';
import 'package:azaman/theme/az_text.dart';
import 'package:azaman/widgets/story_ring.dart';

/// Layout constants are derived, not sprinkled.
abstract final class StoryRailMetrics {
  static const double ring = 64;
  static const double labelGap = AzSpace.xs + 2;
  static const double label = 16;
  static const double verticalPad = AzSpace.md;
  static const double height = ring + labelGap + label + verticalPad * 2; // 108
}

class StoryRailStrip extends ConsumerWidget {
  final void Function(List<StoryGroup> groups, int index) onOpenGroup;
  final VoidCallback onCreate;
  const StoryRailStrip({super.key, required this.onOpenGroup, required this.onCreate});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final feed = ref.watch(storyFeedProvider);
    final myAvatar = ref.watch(authProvider.select((a) => a.user?.profilePictureUrl));
    final groups = feed.valueOrNull ?? const <StoryGroup>[];

    return SizedBox(
      height: StoryRailMetrics.height,
      child: ListView.separated(
        key: const ValueKey('inbox_story_rail_list'),
        scrollDirection: Axis.horizontal,
        padding: const EdgeInsets.symmetric(horizontal: AzSpace.xl, vertical: StoryRailMetrics.verticalPad),
        itemCount: groups.length + 1,
        separatorBuilder: (_, __) => const SizedBox(width: AzSpace.lg - 2),
        itemBuilder: (_, i) {
          if (i == 0) {
            return _RailItem(
              label: 'My story',
              labelColor: colors.textSecondary,
              onTap: onCreate,
              child: Stack(alignment: Alignment.bottomRight, children: [
                StoryRing(avatarUrl: myAvatar, hasUnseenStory: false, isBoosted: false, size: StoryRailMetrics.ring),
                Container(
                  width: 20, height: 20,
                  decoration: BoxDecoration(color: colors.accent, shape: BoxShape.circle, border: Border.all(color: colors.surface, width: 2)),
                  child: Icon(Icons.add, size: 14, color: colors.isDark ? Colors.black : Colors.white),
                ),
              ]),
            );
          }
          final g = groups[i - 1];
          return _RailItem(
            label: g.authorUsername,
            labelColor: g.hasUnseen ? colors.textPrimary : colors.textSecondary,
            onTap: () => onOpenGroup(groups, i - 1),
            child: Hero(
              tag: AzIdentityTag.story(g.authorId),
              child: StoryRing(avatarUrl: g.authorAvatarUrl, hasUnseenStory: g.hasUnseen, isBoosted: g.isBoosted, storyCount: g.stories.length, size: StoryRailMetrics.ring),
            ),
          );
        },
      ),
    );
  }
}

class _RailItem extends StatelessWidget {
  final String label; final Color labelColor; final VoidCallback onTap; final Widget child;
  const _RailItem({required this.label, required this.labelColor, required this.onTap, required this.child});
  @override
  Widget build(BuildContext context) => Semantics(
        button: true, label: '$label story',
        child: GestureDetector(
          onTap: onTap,
          child: SizedBox(
            width: StoryRailMetrics.ring,
            child: Column(mainAxisSize: MainAxisSize.min, children: [
              child,
              const SizedBox(height: StoryRailMetrics.labelGap),
              Text(label, maxLines: 1, overflow: TextOverflow.ellipsis, textAlign: TextAlign.center,
                  style: AzText.caption.copyWith(color: labelColor)),
            ]),
          ),
        ),
      );
}
```

The entrance stagger (`flutter_animate` `.fadeIn(delay: 80*i)`) from the old code is dropped: the rail is now *pulled* into view, so the reveal is the motion.

### 3.2 `lib/widgets/stories/story_rail_compact.dart` — the collapsed affordance

```dart
class StoryRailCompact extends ConsumerWidget {
  final VoidCallback onTap;
  const StoryRailCompact({super.key, required this.onTap});

  static const double height = 40;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final groups = ref.watch(storyFeedProvider.select((f) => f.valueOrNull ?? const <StoryGroup>[]));
    final unseen = groups.where((g) => g.hasUnseen).length;
    final avatars = groups.take(3).toList();

    return Semantics(
      button: true,
      label: unseen > 0 ? '$unseen new stories. Pull down or tap to open stories' : 'Stories. Pull down or tap to open',
      child: GestureDetector(
        key: const ValueKey('inbox_story_rail_compact'),
        onTap: onTap,
        behavior: HitTestBehavior.opaque,
        child: SizedBox(
          height: height,
          child: Row(children: [
            const SizedBox(width: AzSpace.xl),
            SizedBox(
              width: 24.0 + 14.0 * (avatars.length - 1).clamp(0, 2), height: 24,
              child: Stack(children: [
                for (var i = 0; i < avatars.length; i++)
                  Positioned(left: i * 14.0, child: StoryRing(avatarUrl: avatars[i].authorAvatarUrl, hasUnseenStory: avatars[i].hasUnseen, isBoosted: false, size: 24)),
              ]),
            ),
            const SizedBox(width: AzSpace.sm),
            Text(unseen > 0 ? '$unseen new' : 'Stories', style: AzText.caption.copyWith(color: colors.textSecondary)),
            const Spacer(),
            Icon(Icons.keyboard_arrow_down_rounded, size: 18, color: colors.textTertiary),
            const SizedBox(width: AzSpace.xl),
          ]),
        ),
      ),
    );
  }
}
```

When `groups.isEmpty` the compact strip still renders ("Stories") so the rail (with "My story") stays reachable. If the feed errored, the rail shows only "My story" — same as today.

### 3.3 `lib/widgets/stories/story_rail_snap_physics.dart` — the one owner of the snap decision

```dart
import 'package:flutter/physics.dart';
import 'package:flutter/widgets.dart';
import 'package:azaman/experience/motion/az_snap_solver.dart';

/// Snaps the inbox scroll between "rail open" (minScrollExtent, negative) and
/// "rail closed" (0). Positive offsets (the chat list) behave like normal
/// clamping scroll. Deterministic: the target is computed by [AzSnapSolver].
class StoryRailSnapPhysics extends ClampingScrollPhysics {
  const StoryRailSnapPhysics({super.parent, required this.travel});

  /// From `AzMotion.of(context).travel`. When false the snap is instantaneous.
  final bool travel;

  @override
  StoryRailSnapPhysics applyTo(ScrollPhysics? ancestor) =>
      StoryRailSnapPhysics(parent: buildParent(ancestor), travel: travel);

  @override
  Simulation? createBallisticSimulation(ScrollMetrics position, double velocity) {
    final open = position.minScrollExtent; // negative
    if (open >= 0) return super.createBallisticSimulation(position, velocity); // no rail
    final p = position.pixels;
    // Only arbitrate inside the rail band. Above 0 the list behaves normally.
    final inBand = p < 0 || (p == 0 && velocity < 0);
    if (!inBand) return super.createBallisticSimulation(position, velocity);

    final solver = AzSnapSolver(detents: [open, 0], commitFraction: 0.40, flingVelocity: 600);
    final target = solver.resolve(p, velocity);
    if ((target - p).abs() < tolerance.distance) return null;
    if (!travel) return _JumpSimulation(target);
    return ScrollSpringSimulation(spring, p, target, velocity, tolerance: tolerance);
  }

  /// Keep the list from flinging straight through the rail: a fling from the
  /// list that reaches 0 stops there (user pulls again to open).
  @override
  double applyBoundaryConditions(ScrollMetrics position, double value) {
    if (position.pixels > 0 && value < 0) return value; // stop at 0 (closed)
    return super.applyBoundaryConditions(position, value);
  }

  @override
  bool get allowImplicitScrolling => false;
}

class _JumpSimulation extends Simulation {
  _JumpSimulation(this.target);
  final double target;
  @override double x(double time) => target;
  @override double dx(double time) => 0;
  @override bool isDone(double time) => true;
}
```

Notes for the agent:
- `spring` is `ScrollPhysics.spring` (critically damped). If the feel is too soft, construct `SpringDescription.withDampingRatio(mass: 1, stiffness: 350, ratio: 1.0)` — still a constant, documented in the file, never a per-screen magic number elsewhere.
- Semantics: `AzSnapSolver.resolve` treats `velocity > 0` as "toward higher detent" = toward `0` = closing. Dragging down produces negative velocity → opens. Matches brief step 4.

### 3.4 `lib/widgets/stories/inbox_story_rail_sliver.dart` — the sliver before center

```dart
class InboxStoryRailSliver extends StatelessWidget {
  final ScrollController controller;
  final void Function(List<StoryGroup>, int) onOpenGroup;
  final VoidCallback onCreate;
  const InboxStoryRailSliver({super.key, required this.controller, required this.onOpenGroup, required this.onCreate});

  @override
  Widget build(BuildContext context) {
    return SliverToBoxAdapter(
      key: const ValueKey('inbox_story_rail'),
      child: RepaintBoundary(
        child: AnimatedBuilder(
          animation: controller,
          builder: (context, child) {
            double extent = 1;
            if (controller.hasClients) {
              final pos = controller.position;
              final open = pos.minScrollExtent;
              extent = open < 0 ? (pos.pixels / open).clamp(0.0, 1.0) : 1; // 1 = fully open
            }
            return Opacity(
              opacity: Curves.easeOut.transform(extent),
              child: Transform.scale(
                scale: 0.94 + 0.06 * extent,
                alignment: Alignment.topCenter,
                child: child,
              ),
            );
          },
          child: StoryRailStrip(onOpenGroup: onOpenGroup, onCreate: onCreate),
        ),
      ),
    );
  }
}
```

The `child` (the horizontal list) is built once; the builder only rewraps it — nothing inside rebuilds on drag.

### 3.5 `lib/widgets/inbox/inbox_entry.dart` — typed row model (replaces raw maps)

```dart
enum InboxKind { friend, group }
enum InboxSignal { none, moneyIn, moneyOut, request, susuActive, susuConfiguring }

class InboxEntry {
  final InboxKind kind;
  final String id;             // friendshipId or groupId
  final String title;
  final String? avatarUrl;
  final String preview;
  final DateTime? lastAt;
  final int unread;
  final bool pinned;           // from ChatActionsGateway (07) — false until G1
  final bool muted;            // idem
  final InboxSignal signal;
  final int? friendId;         // friend rows
  final GroupSummary? group;   // group rows
  const InboxEntry({...});

  static InboxEntry fromFriend(Map<String, dynamic> friend) {
    final f = friend['friend'] is Map<String, dynamic> ? friend['friend'] as Map<String, dynamic> : const <String, dynamic>{};
    final latest = friend['latestMessage'] is Map<String, dynamic> ? friend['latestMessage'] as Map<String, dynamic> : null;
    final rawAt = latest?['createdAt'] ?? friend['lastMessageTime'];
    final kind = latest?['kind']?.toString();
    final isMe = latest?['isMe'] == true || latest?['senderId']?.toString() == friend['myId']?.toString();
    final signal = switch (kind) {
      'MONEY' || 'TRANSFER' => isMe ? InboxSignal.moneyOut : InboxSignal.moneyIn,
      'REQUEST' => InboxSignal.request,
      _ => InboxSignal.none,
    };
    return InboxEntry(
      kind: InboxKind.friend,
      id: friend['friendshipId']?.toString() ?? friend['id'].toString(),
      title: (f['username'] ?? friend['username'] ?? friend['friendUsername'] ?? 'Unknown').toString(),
      avatarUrl: f['profilePictureUrl']?.toString(),
      preview: (latest?['content'] ?? friend['lastMessage'] ?? '').toString(),
      lastAt: rawAt is DateTime ? rawAt : DateTime.tryParse(rawAt?.toString() ?? ''),
      unread: (friend['unreadCount'] as num?)?.toInt() ?? 0,
      pinned: false, muted: false,
      signal: signal,
      friendId: int.tryParse((f['id'] ?? friend['friendId'] ?? '').toString()),
    );
  }

  static InboxEntry fromGroup(GroupSummary g) => InboxEntry(
        kind: InboxKind.group, id: g.id, title: g.name, avatarUrl: g.avatarUrl,
        preview: g.description ?? '', lastAt: g.updatedAt, unread: 0, pinned: false, muted: false,
        signal: g.isSusuActive ? InboxSignal.susuActive : (g.isSusuConfiguring ? InboxSignal.susuConfiguring : InboxSignal.none),
        group: g,
      );
}
```

The key names (`friendshipId`, `unreadCount`, `latestMessage.kind`) must be confirmed against the live `/friends` payload and the existing `_buildFriendTile` reads (~L1043+). Where a key does not exist, the field defaults (never invents). Keep the existing `_friendSortTime` semantics by reusing the same keys.

### 3.6 `lib/widgets/inbox/inbox_row.dart`

```dart
class InboxRow extends ConsumerWidget {
  final InboxEntry entry;
  final VoidCallback onTap;
  final VoidCallback onLongPress; // row actions (07 §5 sheet, friend variant)
  const InboxRow({super.key, required this.entry, required this.onTap, required this.onLongPress});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final strong = entry.unread > 0;
    return Semantics(
      label: '${entry.title}. ${strong ? '${entry.unread} unread. ' : ''}${entry.preview}',
      child: InkWell(
        onTap: onTap, onLongPress: onLongPress,
        child: Padding(
          padding: const EdgeInsets.symmetric(horizontal: AzSpace.lg, vertical: AzSpace.md),
          child: Row(children: [
            Hero(
              tag: entry.kind == InboxKind.friend && entry.friendId != null
                  ? 'avatar-${entry.friendId}'                 // FriendChatScreen already uses this tag
                  : AzIdentityTag.group(entry.id),
              child: entry.kind == InboxKind.group
                  ? GroupIdentityAvatar(group: entry.group!, size: 48)   // 07 §4 (member stack + Susu ring)
                  : ChatAvatar(imageUrl: entry.avatarUrl, name: entry.title, size: 48),
            ),
            const SizedBox(width: AzSpace.md),
            Expanded(child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [
              Row(children: [
                if (entry.pinned) Icon(Icons.push_pin_rounded, size: 12, color: colors.textTertiary),
                Expanded(child: Text(entry.title, maxLines: 1, overflow: TextOverflow.ellipsis,
                    style: (strong ? AzText.title : AzText.body).copyWith(color: colors.textPrimary))),
                Text(_relative(entry.lastAt), style: AzText.caption.copyWith(color: strong ? colors.accent : colors.textTertiary)),
              ]),
              const SizedBox(height: 2),
              Row(children: [
                _SignalGlyph(signal: entry.signal, colors: colors),
                Expanded(child: Text(entry.preview, maxLines: 1, overflow: TextOverflow.ellipsis,
                    style: AzText.bodyS.copyWith(color: strong ? colors.textPrimary : colors.textSecondary))),
                if (entry.muted) Icon(Icons.notifications_off_outlined, size: 14, color: colors.textTertiary),
                if (strong) ...[const SizedBox(width: AzSpace.sm), ChatUnreadBadge(count: entry.unread, fontSize: 10)],
              ]),
            ])),
          ]),
        ),
      ),
    );
  }

  static String _relative(DateTime? dt) { /* move the existing _formatRelativeTime here verbatim */ }
}

class _SignalGlyph extends StatelessWidget {
  final InboxSignal signal; final AzamanColors colors;
  const _SignalGlyph({required this.signal, required this.colors});
  @override
  Widget build(BuildContext context) {
    final (icon, color) = switch (signal) {
      InboxSignal.moneyIn => (Icons.south_west_rounded, colors.success),
      InboxSignal.moneyOut => (Icons.north_east_rounded, colors.textSecondary),
      InboxSignal.request => (Icons.request_quote_outlined, colors.accent),
      InboxSignal.susuActive => (HugeIconsSolid.coins01, colors.accent),       // verify glyph name
      InboxSignal.susuConfiguring => (HugeIconsSolid.coins01, colors.textTertiary),
      InboxSignal.none => (null, null),
    };
    if (icon == null) return const SizedBox.shrink();
    return Padding(padding: const EdgeInsets.only(right: AzSpace.xs), child: Icon(icon, size: 14, color: color));
  }
}
```

Unread hierarchy = weight + color only (no second badge colour, no bold preview for muted). Transaction context = one 14dp glyph. Not WhatsApp: no green ticks in the inbox, no "You:" prefix; money direction carries that.

---

## 4. The hub rewrite (surgical)

### 4.1 State additions

In `_FriendsHubScreenState`:

```dart
  final ScrollController _inboxScroll = ScrollController();
  static const _centerKey = ValueKey('inbox_center');

  bool get _railOpen => _inboxScroll.hasClients && _inboxScroll.position.pixels <= _inboxScroll.position.minScrollExtent + 0.5;

  void _openRail() {
    if (!_inboxScroll.hasClients) return;
    final travel = AzMotion.of(context).travel;
    final target = _inboxScroll.position.minScrollExtent;
    if (!travel) { _inboxScroll.jumpTo(target); return; }
    _inboxScroll.animateTo(target, duration: MotionTokens.emphasized, curve: MotionTokens.enter);
  }

  void _closeRail() {
    if (!_inboxScroll.hasClients) return;
    final travel = AzMotion.of(context).travel;
    if (!travel) { _inboxScroll.jumpTo(0); return; }
    _inboxScroll.animateTo(0, duration: MotionTokens.standard, curve: MotionTokens.exit);
  }
```

Dispose `_inboxScroll`. The `_toggleSearch` logic is unchanged.

Semantics announcements (brief §3.8): listen once for transitions, not per frame:

```dart
  bool _announcedOpen = false;
  void _onInboxScroll() {
    final open = _railOpen;
    if (open != _announcedOpen) {
      _announcedOpen = open;
      SemanticsService.announce(open ? 'Stories shown' : 'Stories hidden', TextDirection.ltr);
    }
  }
  // initState: _inboxScroll.addListener(_onInboxScroll);
```

### 4.2 Body replacement

Replace the `if (!_isSearching) ...[ story rail Consumer ]` block **and** the trailing `Expanded(child: _isSearching ? _buildSearchResults(...) : _buildChatsTab(...))` with:

```dart
            if (_isSearching) ...[
              const SizedBox(height: 14),
              /* existing search TextField block unchanged */
              const SizedBox(height: 8),
              Expanded(child: _buildSearchResults(colors, provider)),
            ] else
              Expanded(child: _buildInbox(colors, provider)),
```

and add:

```dart
  Widget _buildInbox(AzamanColors colors, FriendProvider provider) {
    final travel = AzMotion.of(context).travel;
    final entries = _entries(provider); // §4.3

    return CustomScrollView(
      key: const ValueKey('inbox_scroll'),
      controller: _inboxScroll,
      center: _centerKey,
      physics: StoryRailSnapPhysics(travel: travel, parent: const AlwaysScrollableScrollPhysics()),
      slivers: [
        // Before center → negative offsets → hidden at rest.
        InboxStoryRailSliver(
          controller: _inboxScroll,
          onOpenGroup: (groups, i) => StoryViewerScreen.open(context, groups: groups, initialGroupIndex: i),
          onCreate: _pickAndCreateStory,
        ),
        // Center → offset 0 at the top of the viewport.
        SliverMainAxisGroup(key: _centerKey, slivers: [
          SliverToBoxAdapter(child: StoryRailCompact(onTap: _openRail)),
          SliverToBoxAdapter(child: _InboxSectionHeader(colors: colors)),
          if (entries.isEmpty)
            SliverFillRemaining(hasScrollBody: false, child: _emptyInbox(colors))
          else
            SliverList.separated(
              itemCount: entries.length,
              separatorBuilder: (_, __) => _softRule(colors), // the existing gradient 1px band, verbatim
              itemBuilder: (context, i) {
                final e = entries[i];
                return InboxRow(
                  entry: e,
                  onTap: () => _openEntry(e),
                  onLongPress: () => _rowActions(e), // 07 §5; until G1: no-op
                );
              },
            ),
          SliverPadding(padding: AzSpace.navClearance),
        ]),
      ],
    );
  }
```

`SliverMainAxisGroup` (Flutter ≥ 3.13) lets one key mark the whole center block. If the project's Flutter is older, use the first center sliver (`StoryRailCompact`'s adapter) as the `center` key and list the rest after it — same offsets.

Important: `CustomScrollView.center` requires `anchor` to remain `0.0` (default) and `shrinkWrap: false`. Do **not** wrap in `RefreshIndicator` (pull-to-refresh and pull-to-reveal are the same gesture; refresh is `friendProvider.refreshAll()` on tab re-entry as today).

### 4.3 Entries (replaces `_ChatListEntry` + the two tile builders)

```dart
  List<InboxEntry> _entries(FriendProvider provider) {
    final groups = ref.watch(groupListProvider).valueOrNull ?? const <GroupSummary>[];
    final list = <InboxEntry>[
      ...provider.friends.map(InboxEntry.fromFriend),
      ...groups.map(InboxEntry.fromGroup),
    ];
    // Pinned first, then recency (pinned state arrives in 07; false until then).
    list.sort((a, b) {
      if (a.pinned != b.pinned) return a.pinned ? -1 : 1;
      final at = a.lastAt, bt = b.lastAt;
      if (at == null && bt == null) return 0;
      if (at == null) return 1;
      if (bt == null) return -1;
      return bt.compareTo(at);
    });
    return list;
  }

  void _openEntry(InboxEntry e) {
    AzamanHaptics.selection();
    if (e.kind == InboxKind.group) {
      pushWithVerticalTransition(context, GroupChatScreen(groupId: e.id));
    } else {
      pushWithVerticalTransition(context, FriendChatScreen(friendshipId: e.id, friendUsername: e.title, friendId: e.friendId ?? 0));
    }
  }
```

(Confirm how `_buildChatsTab` currently reads groups — if it watches a different provider than `groupListProvider`, keep that one.) Delete `_ChatListEntry`, `_GroupChatTile`, `_GroupBubbleAvatar`, `_buildFriendTile`, `_buildFriendsList` once nothing references them (`friend_chat_plus_button_test.dart` and `friends_hub_demo_test.dart` must be checked for finders).

### 4.4 Header tier fix

Anchor: `Text('Inbox', style: TextStyle(color: colors.textPrimary, fontWeight: FontWeight.w800, fontSize: 28, letterSpacing: -0.5))` → `Text('Inbox', style: AzText.titleXl.copyWith(color: colors.textPrimary))`. Keep the `.animate().fadeIn(...)` only if `AzMotion.of(context).travel` (wrap in `AzMotion.pick(context, moving: animated, still: plain)`).

### 4.5 Return path from the viewer

`StoryViewerScreen.open` pushes a route; on pop the inbox is exactly where it was (same `ScrollController`, rail still open). No extra work — this satisfies brief §8.2 (10).

### 4.6 `messages_hub_screen.dart`

Point `app_router.dart:279` to `const FriendsHubScreen()` and mark `MessagesHubScreen` `@Deprecated('Superseded by FriendsHubScreen; remove after C1 soak')`. Do not delete in the same PR.

---

## 5. Group identity and Susu rail hooks (filled by 07)

`GroupIdentityAvatar` (07 §4.1) and the row-actions sheet (07 §5) are imported here. Until G1 lands, C1 ships `GroupIdentityAvatar` as a minimal avatar with the Susu ring and a `_rowActions` no-op — tracked in the PR description as the dependency on G1.

---

## 6. Tests (C1)

`test/screens/friends/inbox_story_rail_test.dart` — pump `FriendsHubScreen` inside `ProviderScope` with overrides: `storyFeedProvider` → fixture of 3 groups, `friendProvider` → 20 friends, `groupListProvider` → 2 groups (one `susuStatus: 'ACTIVE'`).

1. **Closed at rest**: after first frame `controller.offset == 0`; `find.byKey(ValueKey('inbox_story_rail_list'))` is offstage/not hit-testable (check `tester.getRect(...).bottom <= 0` or `Opacity` value 0).
2. **Pull below threshold snaps closed**: `drag(find.byKey('inbox_scroll'), Offset(0, StoryRailMetrics.height * 0.3))`, `pumpAndSettle()` → offset 0.
3. **Pull past threshold snaps open**: drag `height * 0.5` → offset == `minScrollExtent`.
4. **Fling commits**: `fling(..., Offset(0, 30), 900)` → open; from open `fling(..., Offset(0, -30), 900)` → closed.
5. **Compact tap opens**: `tap(find.byKey('inbox_story_rail_compact'))` → open.
6. **List is the owner**: from closed, `drag(Offset(0, -400))` scrolls rows; rail unaffected; drag back down to 0 stops at 0 (no accidental open) — assert `offset == 0` after `pumpAndSettle`.
7. **Reduced motion**: inner `MediaQuery(disableAnimations: true)`; drag `height*0.5` and a single `pump()` → open (no intermediate frames needed).
8. **No duplicate announcements**: mock `SemanticsService` via `TestDefaultBinaryMessenger` or observe `tester.binding.pipelineOwner.semanticsOwner`; open→close→open yields exactly three announcements.
9. **No rebuild of rows on drag**: wrap a row builder in a counter (test-only `InboxRow` key) and assert the count is unchanged after a partial drag.
10. **Tap story → viewer → back restores**: tap a ring, pop, assert `offset == minScrollExtent`.

Goldens (brief §26 #8–10): `inbox_closed`, `inbox_mid_pull` (jump controller to `minScrollExtent/2`), `inbox_open` via `pumpGoldenSurface`.

---

## 7. Acceptance (on device)

- Inbox opens with no rail; compact strip reads "N new" when unseen stories exist.
- Pull from the very top: rail peels in continuously, release past 40% or any quick flick → opens; otherwise closes. Partial positions never stick.
- While open, scrolling the list upward first closes the rail (natural), then scrolls rows; scrolling back to the top stops at closed — pull again to reopen.
- Tapping a ring opens the viewer; back returns to the same rail state and list position.
- Reduced motion: rail toggles instantly; VoiceOver/TalkBack announces "Stories shown/hidden" once per change.
- No jank at 360dp with 200 rows: Flutter DevTools shows only the rail layer repainting during the pull.