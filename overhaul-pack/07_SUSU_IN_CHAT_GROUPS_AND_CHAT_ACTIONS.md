# 07 — Susu Social Proof in Chat, Group Identity, Group Info, and the Real Personal-Chat Action Menu

> **Reconciled against live code — frontend `main` `6810007` (after #135 storefront shell, #136 Add Cash/typography), backend `main`, planning `main`.**
> Every identifier below is tagged **EXISTING** (`path:line` on live main) or **NEW** (introduced by this document). Anchors were re-verified with `git grep` on the live tree; this doc supersedes the `30188ef` snapshot version in PR #60. Review points from the PR #60 review that this document resolves:

| Review point | Resolution in this document |
|---|---|
| Susu "every week" hardcoded | §3.2 `SusuSocialView.cadenceLabel` from EXISTING `SusuFrequency`; copy assembled from the typed view |
| GHS conversion implicit | §3.1 `contributionGhs` + `rateSource` computed in `derive(...)` from injected EXISTING `SusuSuppliedRate`; widgets never convert |
| Notification anchor wrong | §5.1 re-anchored on `main.dart:745 _showSocketNotificationBanner` / `InAppPushBanner.show`, `_handleForeground`, `friend_provider.dart:65`; truthful "while the app is open" copy |
| Anchors | EXISTING: `friend_chat_screen.dart:605–608` (`Icons.more_vert → _openChatProfile`), `_openSearch :285`, `_openChatProfile :476`; `group_chat/group_chat_screen.dart` `_SusuEventCard :409`, `_SusuBanner :436`; `group_chat/group_profile_screen.dart` `_InitiateCta :386`, `_ActiveSusuBadge :576`; `premiumChatProvider` family (`premium_chat_provider.dart:78`), `ChatContext {friend, trade, group, ticket}` |
| Capability truth | report/block/archive/clear-remote stay `AzUnsupported` with exact endpoints; `removeFriend` is never aliased to block |

Implements brief §12 (Susu in chat), §13 (three-dot menu), §14 (group chat), plus the Susu/row-action seams left by 04. Depends on 01 (gateways, `AzIntent`, `AzIdentityTag`) and 04 (`InboxRow`, `GroupIdentityAvatar` slot).

Primary files:
- `lib/screens/group_chat/group_chat_screen.dart` (`_SusuBanner` → `SusuStatusChip`; header identity)
- `lib/screens/group_chat/group_profile_screen.dart` (member area → participants vs observers, orbit, explainer)
- `lib/screens/friends/friend_chat_screen.dart` (`more_vert` → `ChatActionsSheet`; in-conversation search)
- `lib/screens/chat_profile_screen.dart` (gains the action rows)
- new `lib/experience/gateways/susu_membership_gateway.dart`, `chat_actions_gateway.dart`
- new `lib/widgets/susu/{susu_status_chip,susu_member_orbit,susu_explainer_section}.dart`
- new `lib/widgets/inbox/group_identity_avatar.dart`, `lib/widgets/chat/actions/{chat_actions_sheet,conversation_search_bar,mute_duration_sheet}.dart`

Untouched: `InitiateSusuSheet`, `SusuDashboardScreen`, `SusuService` financial calls, contracts, vouching, PoR, `premiumChatProvider` send paths.

---

## 1. Susu membership derivation (the truthful core)

`GroupSummary.members` (group) and the Susu's members (`susuMembersProvider(susuId)` → `List<SusuMemberView>` with `userId`, `status`, `cycleSlot`; or `susuInitiationStatusProvider(groupId).members` while configuring) are two different sets. The brief's rule: a group member who is not an active Susu member is an **observer** — shown, never counted as a contributor.

### 1.1 `lib/experience/gateways/susu_membership_gateway.dart`

```dart
enum SusuRole { participant, pendingParticipant, observer }

class SusuSeat {
  final GroupMember member;          // identity from the group (always present)
  final SusuRole role;
  final SusuMemberStatus? susuStatus; // null for observers
  final int? cycleSlot;               // payout order when known
  final bool isMe;
  final bool isGroupAdmin;
  const SusuSeat({required this.member, required this.role, this.susuStatus, this.cycleSlot, required this.isMe, required this.isGroupAdmin});
}

class SusuSocialView {
  final String? susuId;
  final SusuStatus status;                 // unknown when the group has no Susu
  final List<SusuSeat> seats;              // participants first (by cycleSlot), then pending, then observers
  final SusuCycleView? currentCycle;       // from susuCyclesProvider when active
  final double? contributionUsdc; final SusuFrequency? frequency; final int? totalCycles;
  /// GHS display amount, derived ONCE here from EXISTING `susuSuppliedRateProvider`
  /// (`SusuSuppliedRate{usdcToGhs, source}`, lib/providers/susu_provider.dart:501).
  /// `null` when `contributionUsdc` is null or `rateSource == 'UNAVAILABLE'`.
  /// Widgets never convert or fetch a rate themselves.
  final double? contributionGhs; final String rateSource;
  final bool iCanJoin;                     // eligibility from invite/initiation state
  const SusuSocialView({...});

  /// Human cadence from EXISTING `SusuFrequency {daily, weekly, biweekly, monthly, unknown}`
  /// (lib/models/susu_model.dart:120). `null` for unknown → copy omits the cadence.
  String? get cadenceLabel => switch (frequency) {
        SusuFrequency.daily => 'every day', SusuFrequency.weekly => 'every week',
        SusuFrequency.biweekly => 'every two weeks', SusuFrequency.monthly => 'every month',
        SusuFrequency.unknown || null => null,
      };

  int get participantCount => seats.where((s) => s.role == SusuRole.participant).length;
  int get observerCount => seats.where((s) => s.role == SusuRole.observer).length;
  bool get hasSusu => susuId != null && status != SusuStatus.unknown;
}

abstract interface class SusuMembershipGateway {
  /// Pure derivation — unit-testable without network.
  SusuSocialView derive({
    required GroupSummary group,
    required int myUserId,
    SusuDetail? detail,                    // active/completed
    SusuInitiationStatus? initiation,      // configuring
    List<SusuCycleView> cycles = const [],
    required SusuSuppliedRate rate,        // EXISTING type; injected so derivation stays pure & testable
  });
}

class DefaultSusuMembershipGateway implements SusuMembershipGateway {
  const DefaultSusuMembershipGateway();

  @override
  SusuSocialView derive({required GroupSummary group, required int myUserId, SusuDetail? detail, SusuInitiationStatus? initiation, List<SusuCycleView> cycles = const []}) {
    final byUser = <int, SusuMemberView>{for (final m in detail?.members ?? const <SusuMemberView>[]) m.userId: m};
    final initByUser = <int, SusuInitiationMember>{for (final m in initiation?.members ?? const <SusuInitiationMember>[]) m.userId: m};

    SusuSeat seat(GroupMember gm) {
      final sv = byUser[gm.userId];
      final iv = initByUser[gm.userId];
      final status = sv?.status ?? SusuMemberStatus.fromString(iv?.susuMemberStatus);
      final role = switch (status) {
        SusuMemberStatus.active => SusuRole.participant,
        SusuMemberStatus.pendingVouch || SusuMemberStatus.pendingContract => SusuRole.pendingParticipant,
        _ => SusuRole.observer,
      };
      return SusuSeat(member: gm, role: role, susuStatus: role == SusuRole.observer ? null : status, cycleSlot: sv?.cycleSlot, isMe: gm.userId == myUserId, isGroupAdmin: gm.role == 'ADMIN');
    }

    final seats = group.members.where((m) => m.removedAt == null).map(seat).toList()
      ..sort((a, b) {
        if (a.role != b.role) return a.role.index.compareTo(b.role.index);
        return (a.cycleSlot ?? 1 << 20).compareTo(b.cycleSlot ?? 1 << 20);
      });

    final status = detail?.status ?? SusuStatus.fromString(group.susuStatus);
    final current = cycles.where((c) => c.status == SusuCycleStatus.collecting || c.status == SusuCycleStatus.collectingGrace).firstOrNull
        ?? cycles.where((c) => c.status == SusuCycleStatus.pending).firstOrNull;

    final mine = seats.where((s) => s.isMe).firstOrNull;
    final iCanJoin = mine != null && mine.role == SusuRole.observer && status == SusuStatus.configuring; // joining mid-cycle is not a thing

    return SusuSocialView(
      susuId: group.susuGroupId, status: status, seats: seats, currentCycle: current,
      contributionUsdc: detail?.contributionUsdc, frequency: detail?.frequency, totalCycles: detail?.totalCycles,
      iCanJoin: iCanJoin,
    );
  }
}

final susuMembershipGatewayProvider = Provider<SusuMembershipGateway>((_) => const DefaultSusuMembershipGateway());

/// Composes the existing providers. Autodispose; never fetches on its own.
final susuSocialViewProvider = Provider.autoDispose.family<AsyncValue<SusuSocialView>, String>((ref, groupId) {
  final group = ref.watch(groupDetailProvider(groupId));
  final myId = int.tryParse(ref.watch(authProvider).user?.id.toString() ?? '') ?? -1;
  return group.whenData((g) {
    final init = g.isSusuConfiguring ? ref.watch(susuInitiationStatusProvider(groupId)).valueOrNull : null;
    final detail = (g.susuGroupId != null && !g.isSusuConfiguring) ? ref.watch(susuDetailV2Provider(g.susuGroupId!)).valueOrNull : null;
    final cycles = g.isSusuActive ? (ref.watch(susuCyclesProvider(g.susuGroupId!)).valueOrNull ?? const <SusuCycleView>[]) : const <SusuCycleView>[];
    return ref.read(susuMembershipGatewayProvider).derive(group: g, myUserId: myId, detail: detail, initiation: init, cycles: cycles);
  });
});
```

Privacy rule: the social view exposes `contributionUsdc/frequency/totalCycles/currentCycle` only if the detail provider returned them for *this* user — i.e. the server already authorises. Observers who are not permitted to see amounts get `null` from the API and the UI omits the row. The client never computes totals ("pot size") from `contribution × participants` — that is derived money truth (brief §12.4).

Verify `groupDetailProvider`, `susuDetailV2Provider`, `susuCyclesProvider` signatures in `group_chat_provider.dart` / `susu_provider.dart`; `authProvider.user.id` type as used in `friend_chat_screen.dart:552`.

---

## 2. Group chat screen

### 2.1 `SusuStatusChip` replaces `_SusuBanner` (anchor `group_chat_screen.dart:436–470`)

Today: a full-width banner "Active Susu cycle in progress" → `SusuDashboardScreen`. New: a compact chip rail under the app bar, visible in `configuring`/`active`/`frozenDispute`, hidden otherwise.

```dart
class SusuStatusChip extends ConsumerWidget {
  final String groupId;
  const SusuStatusChip({super.key, required this.groupId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    final view = ref.watch(susuSocialViewProvider(groupId)).valueOrNull;
    if (view == null || !view.hasSusu || view.status == SusuStatus.completed || view.status == SusuStatus.cancelled) return const SizedBox.shrink();

    final (label, tone) = switch (view.status) {
      SusuStatus.configuring => ('Susu starting · ${view.participantCount} in', colors.textSecondary),
      SusuStatus.active => (_activeLabel(view), colors.accent),
      SusuStatus.frozenDispute => ('Susu paused', colors.warning),
      _ => ('Susu', colors.textSecondary),
    };

    return Padding(
      padding: const EdgeInsets.fromLTRB(AzSpace.lg, AzSpace.sm, AzSpace.lg, 0),
      child: Semantics(button: true, label: label, child: ScaleTap(
        onTap: () {
          AzamanHaptics.selection();
          if (view.status == SusuStatus.configuring) pushWithVerticalTransition(context, GroupProfileScreen(groupId: groupId));
          else pushWithVerticalTransition(context, SusuDashboardScreen(susuId: view.susuId!));
        },
        child: Container(
          key: const ValueKey('group_susu_chip'),
          height: 32, padding: const EdgeInsets.symmetric(horizontal: AzSpace.md),
          decoration: BoxDecoration(color: colors.softSurface, borderRadius: BorderRadius.circular(AzRadius.pill)),
          child: Row(mainAxisSize: MainAxisSize.min, children: [
            Icon(HugeIconsSolid.coins01, size: 14, color: tone),
            const SizedBox(width: AzSpace.sm),
            Text(label, style: AzText.label.copyWith(color: tone), maxLines: 1, overflow: TextOverflow.ellipsis),
            const SizedBox(width: AzSpace.xs),
            Icon(Icons.chevron_right_rounded, size: 14, color: colors.textTertiary),
          ]),
        ),
      )),
    );
  }

  String _activeLabel(SusuSocialView v) {
    final c = v.currentCycle;
    if (c == null || v.totalCycles == null) return 'Susu active · ${v.participantCount} members';
    final when = _relativeDay(c.scheduledRunAt); // 'today' / 'Fri' / '12 Oct' via intl
    return 'Round ${c.cycleNumber}/${v.totalCycles} · $when';
  }
}
```

No amounts in the chip. The "next contribution/payout" wording appears only when `currentCycle` exists (authoritative).

### 2.2 Header identity

Anchor `group_chat_screen.dart:165–175` (`title: groupAsync.when(... GestureDetector → GroupProfileScreen)`). Wrap the avatar in `Hero(tag: AzIdentityTag.group(groupId))` and use `GroupIdentityAvatar(group: g, size: 36)` (§4.1) so inbox → chat → profile keep one identity. Subtitle line: `'${g.members.length} members'` plus `' · Susu'` when `isSusuEnabled` (no counts beyond that in the header).

### 2.3 Group three-dot

Add `actions: [IconButton(Icons.more_vert → GroupActionsSheet)]` using the same `ChatActionsSheet` (§5) with `ChatActionScope.group` — capabilities filter what shows (search, mute, media, pinned, group info, leave if supported, report).

---

## 3. Group profile — participants vs observers, orbit, explainer

Anchor `group_profile_screen.dart` member list builder (~L108–L160, `'MEMBERS · ${group.members.length}'` header + `group.members.map((gm) {...})`). Replace that section with three widgets fed by `susuSocialViewProvider(groupId)`:

```dart
final social = ref.watch(susuSocialViewProvider(groupId)).valueOrNull;
...
if (social != null && social.hasSusu) ...[
  SusuMemberOrbit(view: social, colors: colors),
  const SizedBox(height: AzSpace.lg),
  SusuExplainerSection(view: social, colors: colors, onJoin: social.iCanJoin ? () => _join(context, ref, social) : null),
  const SizedBox(height: AzSpace.xl),
],
_MemberGroups(view: social, group: group, colors: colors, iAmAdmin: iAmAdmin),
```

### 3.1 `SusuMemberOrbit` — `lib/widgets/susu/susu_member_orbit.dart`

Participants on a ring ordered by `cycleSlot` (payout order), the current payout seat highlighted, observers as a muted stacked row beneath. Static geometry; the only motion is a 450 ms draw-in of the ring on first build (reduced motion → none).

```dart
class SusuMemberOrbit extends StatelessWidget {
  final SusuSocialView view; final AzamanColors colors;
  const SusuMemberOrbit({super.key, required this.view, required this.colors});

  @override
  Widget build(BuildContext context) {
    final participants = view.seats.where((s) => s.role != SusuRole.observer).toList();
    final observers = view.seats.where((s) => s.role == SusuRole.observer).toList();
    final payoutUser = view.currentCycle?.payoutUserId;
    final travel = AzMotion.of(context).travel;
    final progress = (view.totalCycles != null && view.currentCycle != null) ? (view.currentCycle!.cycleNumber - 1) / view.totalCycles! : 0.0;

    return RepaintBoundary(child: Column(children: [
      SizedBox(
        height: 220,
        child: TweenAnimationBuilder<double>(
          tween: Tween(begin: travel ? 0 : 1, end: 1),
          duration: travel ? MotionTokens.spatial : Duration.zero, curve: MotionTokens.enter,
          builder: (context, t, _) => LayoutBuilder(builder: (context, c) {
            final center = Offset(c.maxWidth / 2, 110);
            const r = 80.0;
            return Stack(children: [
              CustomPaint(size: Size(c.maxWidth, 220), painter: _OrbitPainter(center: center, radius: r, progress: progress * t, track: colors.softSurface, accent: colors.accent)),
              for (var i = 0; i < participants.length; i++)
                _seatAt(center, r, i, participants.length, t, participants[i], participants[i].member.userId == payoutUser),
              Positioned.fill(child: Center(child: Column(mainAxisSize: MainAxisSize.min, children: [
                Text('${participants.length}', style: AzText.titleXl.copyWith(color: colors.textPrimary)),
                Text('in the Susu', style: AzText.caption.copyWith(color: colors.textSecondary)),
              ]))),
            ]);
          }),
        ),
      ),
      if (observers.isNotEmpty) Padding(
        padding: const EdgeInsets.only(top: AzSpace.sm),
        child: Row(mainAxisAlignment: MainAxisAlignment.center, children: [
          SizedBox(width: 24.0 + 14.0 * (observers.length.clamp(1, 4) - 1), height: 24, child: Stack(children: [
            for (var i = 0; i < observers.length.clamp(0, 4); i++)
              Positioned(left: i * 14.0, child: Opacity(opacity: .7, child: ChatAvatar(imageUrl: observers[i].member.profilePictureUrl, name: observers[i].member.username ?? '?', size: 24))),
          ])),
          const SizedBox(width: AzSpace.sm),
          Text('${observers.length} watching', style: AzText.caption.copyWith(color: colors.textTertiary)),
        ]),
      ),
    ]));
  }

  Widget _seatAt(Offset c, double r, int i, int n, double t, SusuSeat s, bool isPayout) {
    final a = -math.pi / 2 + 2 * math.pi * i / n;
    final p = c + Offset(math.cos(a), math.sin(a)) * (r * t);
    final size = isPayout ? 44.0 : 36.0;
    return Positioned(
      left: p.dx - size / 2, top: p.dy - size / 2,
      child: Semantics(
        label: '${s.member.username ?? 'Member'}${s.cycleSlot != null ? ', payout ${s.cycleSlot}' : ''}${isPayout ? ', receives this round' : ''}${s.role == SusuRole.pendingParticipant ? ', pending' : ''}',
        child: Opacity(opacity: s.role == SusuRole.pendingParticipant ? .55 : 1, child: Stack(clipBehavior: Clip.none, children: [
          Container(
            decoration: isPayout ? BoxDecoration(shape: BoxShape.circle, border: Border.all(color: colors.accent, width: 2)) : null,
            child: ChatAvatar(imageUrl: s.member.profilePictureUrl, name: s.member.username ?? '?', size: size),
          ),
          if (s.cycleSlot != null) Positioned(right: -4, bottom: -4, child: Container(
            width: 18, height: 18, alignment: Alignment.center,
            decoration: BoxDecoration(color: colors.card, shape: BoxShape.circle),
            child: Text('${s.cycleSlot}', style: AzText.caption.copyWith(color: colors.textPrimary, fontSize: 10)),
          )),
        ])),
      ),
    );
  }
}

class _OrbitPainter extends CustomPainter {
  final Offset center; final double radius; final double progress; final Color track; final Color accent;
  const _OrbitPainter({required this.center, required this.radius, required this.progress, required this.track, required this.accent});
  @override void paint(Canvas canvas, Size size) {
    final p = Paint()..style = PaintingStyle.stroke..strokeWidth = 3..strokeCap = StrokeCap.round;
    canvas.drawCircle(center, radius, p..color = track);
    if (progress > 0) canvas.drawArc(Rect.fromCircle(center: center, radius: radius), -math.pi / 2, 2 * math.pi * progress, false, p..color = accent);
  }
  @override bool shouldRepaint(_OrbitPainter o) => o.progress != progress || o.center != center || o.radius != radius;
}
```

With > 12 participants the ring shows 12 and a "+N" seat; the full list is below. No amounts anywhere in the orbit. The existing `susu_wheel.dart` (`SusuWheel`) is for the dashboard and remains; the orbit is a social summary, not a replacement — note this in the map.

### 3.2 `SusuExplainerSection` — "How this Susu works"

Rows render **only** when the value is non-null:

```dart
class SusuExplainerSection extends ConsumerWidget {
  final SusuSocialView view; final AzamanColors colors; final VoidCallback? onJoin;
  ...
  // rows (hairline separated, no cards):
  //  Contribution   AzMoney from contributionUsdc + susuSuppliedRateProvider cedis (existing conversion pattern in _InitiationSummary)
  //  Frequency      frequency.label
  //  Round          'currentCycle.cycleNumber of totalCycles' when both present
  //  Members        '${participantCount} contributing · ${observerCount} watching'
  //  Payout order   'By slot' + first 3 names, when any cycleSlot exists
  //  Status         status label (configuring/active/paused)
  //  Join           filled button when onJoin != null; else, if I'm an observer and status != configuring: 'Joining opens with the next Susu' (truthful)
}
```

Copy avoids finance jargon and is assembled from the typed view, never hardcoded: `"Everyone puts in ${ghs != null ? 'GH₵ $ghs' : 'the same amount'}${cadence != null ? ' $cadence' : ''}; each round one member receives the pot."` — `ghs = view.contributionGhs` (null when the rate source is `UNAVAILABLE`), `cadence = view.cadenceLabel` (null for `SusuFrequency.unknown`). A monthly Susu reads "every month"; an unknown cadence shows none. Test all five frequencies and the no-rate case in `susu_explainer_section_test.dart`.

Join routes to the **existing** invite acceptance (`susuActionsProvider` / `SusuService.acceptInvite`) when an invite for me exists; if none exists the button is not shown (no invented join endpoint).

### 3.3 `_MemberGroups`

Two plain sections replacing the single list: "In the Susu · N" (seats with role ≠ observer, each row showing name, slot badge, `verification_chip.dart` as today, pending state) and "Also in the group · N" (observers). Admin controls (`_addMember`) stay with the admin check as today. When the group has no Susu, a single "Members · N" section renders exactly as before.

---

## 4. Group identity

### 4.1 `lib/widgets/inbox/group_identity_avatar.dart`

```dart
class GroupIdentityAvatar extends ConsumerWidget {
  final GroupSummary group; final double size;
  const GroupIdentityAvatar({super.key, required this.group, required this.size});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final colors = ref.watch(themeProvider).colors;
    Widget core;
    if (group.avatarUrl != null) {
      core = ChatAvatar(imageUrl: group.avatarUrl, name: group.name, size: size);
    } else {
      final heads = group.members.where((m) => m.removedAt == null).take(3).toList();
      core = SizedBox(width: size, height: size, child: Stack(children: [
        for (var i = 0; i < heads.length; i++)
          Positioned(left: (i % 2) * size * .35, top: (i ~/ 2) * size * .35,
            child: ChatAvatar(imageUrl: heads[i].profilePictureUrl, name: heads[i].username ?? '?', size: size * .6)),
      ]));
    }
    if (!group.isSusuEnabled) return core;
    return Container(
      padding: const EdgeInsets.all(2),
      decoration: BoxDecoration(shape: BoxShape.circle, border: Border.all(color: group.isSusuActive ? colors.accent : colors.textTertiary, width: 2)),
      child: core,
    );
  }
}
```

The Susu ring is the one group marker allowed on avatars; it is the same visual the orbit uses for the payout seat, so the language is consistent. `_GroupBubbleAvatar` in the hub is replaced by this.

---

## 5. Personal chat — the real three-dot

### 5.1 `lib/experience/gateways/chat_actions_gateway.dart`

```dart
enum ChatActionCapability { search, mute, mediaFiles, pin, markUnread, report, block, clearLocal, clearRemote, archive, profile, sharedFinance }
enum ChatActionScope { friend, group }

class MuteState { final DateTime? until; const MuteState({this.until}); bool get isMuted => until == null ? false : until!.isAfter(DateTime.now()); }

class ConversationSearchHit { final String messageId; final int indexInLoaded; final String snippet; const ConversationSearchHit(...); }

abstract interface class ChatActionsGateway {
  Set<ChatActionCapability> capabilitiesFor(ChatActionScope scope);

  // Local preference state (SharedPreferences) — implemented now.
  Stream<MuteState> mute(String conversationId);
  Future<void> setMute(String conversationId, Duration? forDuration);   // null → unmute
  Stream<bool> pinned(String conversationId);
  Future<void> setPinned(String conversationId, bool value);
  Stream<bool> markedUnread(String conversationId);
  Future<void> setMarkedUnread(String conversationId, bool value);

  // Searches already-loaded messages; remote search via MessageActionService.searchMessages when available.
  List<ConversationSearchHit> searchLoaded(List<ChatMessage> messages, String query);
  Future<AzGatewayResult<List<ConversationSearchHit>>> searchRemote(String conversationId, String query);

  // Server-backed — unsupported until endpoints exist.
  Future<AzGatewayResult<void>> report(String conversationId, {required String reason, String? details});
  Future<AzGatewayResult<void>> block(int userId);
  Future<AzGatewayResult<void>> unblock(int userId);
  Future<AzGatewayResult<void>> clearRemote(String conversationId);
  Future<AzGatewayResult<void>> archive(String conversationId, bool value);
}
```

Default adapter `PrefsChatActionsGateway`:
- `capabilitiesFor(friend)` = `{search, mute, mediaFiles, pin, markUnread, clearLocal, profile, sharedFinance}`; group adds nothing extra, removes `sharedFinance`/`block`.
- Mute/pin/unread: `SharedPreferences` keys `az_chat_mute_<id>` (epoch ms), `az_chat_pin_<id>`, `az_chat_unread_<id>`; streams via a `StreamController.broadcast` per key. The inbox (`InboxEntry.pinned/muted` in 04) reads these through `ref.watch(chatPrefsProvider)`.
- Mute suppression at the **real presentation boundaries** (verified on live main — `push_notification_service.dart` does not display chat notifications):
  1. `lib/main.dart:745 _showSocketNotificationBanner` → `InAppPushBanner.show` (`lib/widgets/in_app_push_banner.dart:26`), fed by `socketService.onNewNotification` (`new_notification`). Before `InAppPushBanner.show`, resolve the chat id from the payload (`friendshipId` / `groupId` / `action`) and skip when `chatPrefs.isMuted(id)`; keep the unread-count increment.
  2. `lib/services/push_notification_service.dart:68 → _handleForeground(RemoteMessage)` (FCM while foregrounded): same check on `message.data` before any foreground presentation. Background FCM delivery is OS-side and **cannot** be suppressed client-side → the sheet copy is "Muted on this device while the app is open"; server-side mute is `AzUnsupported('PUT /friends/:id/mute')`.
  3. `lib/providers/friend_provider.dart:65 _handleFriendMessage` (socket `friend_message`) bumps `unreadCount` and shows nothing — leave the bump; muted threads still count unread, they are just silent (matches the "mute" semantics users expect).
  Add `test/widgets/chat/mute_suppression_test.dart` driving a fake socket payload through the shell handler with a muted and an unmuted id.
- `report/block/unblock/clearRemote/archive` → `AzUnsupported` with the exact endpoint needed in the message (`POST /friends/:id/report`, `POST /users/:id/block`, `DELETE /friends/chat/:id/messages`, `PUT /friends/:id/archive`). `removeFriend` (`DELETE /friends/:id`) exists and is **not** block — do not alias them.
- `clearLocal`: clears the in-memory `PremiumChatState.messages` via a new `PremiumChatNotifier.clearLocalHistory()` (state = copyWith(messages: [], hasMore: false)) — the confirmation copy says "Removes messages from this device only. The other person still has them." (brief §13.4).

### 5.2 `ChatActionsSheet` — replaces `friend_chat_screen.dart:606`

Anchor:

```dart
          IconButton(
            icon: Icon(Icons.more_vert, color: c.textPrimary),
            onPressed: _openChatProfile,
          ),
```

Replacement:

```dart
          IconButton(
            key: const ValueKey('friend_chat_more'),
            icon: Icon(Icons.more_vert, color: c.textPrimary),
            tooltip: 'More',
            onPressed: () => ChatActionsSheet.show(context, ref,
                scope: ChatActionScope.friend, conversationId: widget.friendshipId, peerUserId: widget.friendId, peerName: widget.friendUsername,
                onSearch: _startConversationSearch, onOpenProfile: _openChatProfile),
          ),
```

```dart
class ChatActionsSheet {
  static Future<void> show(BuildContext context, WidgetRef ref, {required ChatActionScope scope, required String conversationId, int? peerUserId, required String peerName, required VoidCallback onSearch, required VoidCallback onOpenProfile}) {
    final gw = ref.read(chatActionsGatewayProvider);
    final caps = gw.capabilitiesFor(scope);
    return AzamanSheet.showPanel<void>(context, builder: (ctx, sc) => _ChatActionsBody(caps: caps, ...));
  }
}
```

Rows (each only if its capability is present), in this order, with `AzIntent` for haptics and destructive styling:

| Row | Capability | Action |
|-----|-----------|--------|
| Search in conversation | search | pop sheet → `onSearch()` |
| Mute / Unmute (`MuteState` aware) | mute | `MuteDurationSheet` (8h · 1 week · Always) → `setMute` |
| Media, files & links | mediaFiles | `onOpenProfile()` (the existing tabs) |
| Pin / Unpin | pin | `setPinned` |
| Mark as unread | markUnread | `setMarkedUnread(true)` + pop to inbox (`Navigator.pop`) |
| Shared money | sharedFinance | `ChatProfileScreen` Receipts tab (existing) — label "Receipts & transfers" |
| View profile | profile | `onOpenProfile()` |
| Clear chat on this device | clearLocal | destructive confirm (scope copy above) → `clearLocalHistory()` |
| Report | report | destructive confirm with reason picker → `report()`; copy: "Azaman reviews reports within 48h" **only if the backend confirms an SLA; otherwise 'Our team will review this report.'** |
| Block | block | destructive confirm: "They won't be able to message or send you money. You can unblock from their profile." → `block()`; on `AzOk` the input bar is replaced by a "You blocked X · Unblock" strip |

Since `report`/`block`/`archive`/`clearRemote` are unsupported today, those rows **do not render** — the menu is honest. When the backend adds them, flipping the capability set lights the rows with no UI change.

### 5.3 In-conversation search — `conversation_search_bar.dart`

Replaces `_openSearch()` (which pushes the global `MessageSearchScreen`) for in-chat search; the global screen stays reachable from the inbox.

State in `_FriendChatScreenState`:

```dart
  bool _searching = false;
  final _searchCtrl = TextEditingController();
  List<ConversationSearchHit> _hits = const [];
  int _hitIndex = -1;
  final Map<String, GlobalKey> _msgKeys = {}; // only for messages that are hits

  void _startConversationSearch() => setState(() => _searching = true);

  void _onQuery(String q) {
    final gw = ref.read(chatActionsGatewayProvider);
    final msgs = ref.read(premiumChatProvider(params)).messages;
    final hits = gw.searchLoaded(msgs, q);
    setState(() { _hits = hits; _hitIndex = hits.isEmpty ? -1 : 0; });
    if (hits.isNotEmpty) _jumpTo(0);
    // Remote completion for messages not loaded yet (if supported):
    if (q.length >= 2 && gw.capabilitiesFor(ChatActionScope.friend).contains(ChatActionCapability.search)) {
      _remoteSearch = gw.searchRemote(widget.friendshipId, q).then((r) { if (!mounted || _searchCtrl.text != q) return; /* merge hits by messageId */ });
    }
  }

  void _jumpTo(int i) {
    if (i < 0 || i >= _hits.length) return;
    _hitIndex = i;
    final hit = _hits[i];
    // Messages render reversed: item index == position in state.messages.
    _scrollController.animateTo(/* use Scrollable.ensureVisible on the key if built; else estimate via itemExtent-less approach: */ 0,
        duration: Duration.zero, curve: Curves.linear);
    final key = _msgKeys[hit.messageId];
    if (key?.currentContext != null) {
      Scrollable.ensureVisible(key!.currentContext!, alignment: .5, duration: AzMotion.duration(context, MotionTokens.standard), curve: MotionTokens.enter);
    } else {
      // Not built: scroll by index. With `reverse: true` and variable heights, use the
      // existing ListView.builder with `findChildIndexCallback` + a one-off `jumpTo`
      // to the max extent then ensureVisible after the frame (documented limitation:
      // very old hits require loadMessages(loadMore: true) until the id is present).
    }
    setState(() {});
  }
```

Bar UI: replaces the `AppBar` title while `_searching` — `TextField` (autofocus), "3 of 12", ▲ ▼ buttons (`_jumpTo(_hitIndex ± 1)`), close. Bubble highlight: `PremiumMessageBubble` gets an optional `highlightQuery: String?` and `isCurrentHit: bool` → the bubble wraps matching substrings in `TextSpan(backgroundColor: colors.accent.withValues(alpha: .3))`; current hit gets a 2dp accent outline. Keys: `conversation_search_bar`, `conversation_search_count`.

`searchLoaded` default implementation: case-insensitive `contains` over `ChatMessage.text` for `MessageKind.text` and over `replyToText`; returns hits newest-first to match list order. `searchRemote` calls `MessageActionService.searchMessages(query:…)` and filters by `friendshipId`/contextId if the service returns cross-conversation results (check its return shape at `message_search_screen.dart:33`).

Cancellation: `_remoteSearch` result is ignored when the query changed or the widget unmounted; `dispose` clears controllers.

### 5.4 Chat details/profile page (`chat_profile_screen.dart`)

Add an "Actions" section above the tabs with the same rows as the sheet (built from the same `_ChatActionsBody` row list in `inline: true` mode) so mute/pin/block state is visible in one place. Trust breakdown and nickname stay.

---

## 6. Group chat as a social space (brief §14)

- Header: `GroupIdentityAvatar` + name + "N members · Susu" line (§2.2).
- `SusuStatusChip` rail (§2.1).
- Pinned messages: only if the backend exposes them (`ChatMessage.metadata['pinned']`? — check); otherwise no pinned UI.
- Shared media/search/mute: `ChatActionsSheet(scope: group)`.
- Roles: `GroupMember.role` already drives admin controls in the profile; member rows show an "Admin" caption as today.
- Group story context: if any member has an unseen story (`storyFeedProvider` groups ∩ member userIds), the profile header shows a small `StoryRing(size: 20)` cluster → `StoryViewerScreen.open` with those groups. Nothing when none.

---

## 7. Tests (G1)

- `test/experience/susu_membership_gateway_test.dart`: participant/pending/observer classification from `SusuDetail` and from `SusuInitiationStatus`; ordering by slot; `iCanJoin` only for observers while configuring; removed members excluded; counts truthful (observers never counted).
- `test/widgets/susu/susu_member_orbit_test.dart`: N seats positioned; payout seat highlighted; >12 → "+N"; semantics labels; reduced motion renders final geometry on first frame.
- `test/widgets/susu/susu_explainer_section_test.dart`: rows absent when values null; join button absent when not eligible.
- `test/widgets/chat/chat_actions_sheet_test.dart`: rows match capability set (override gateway with a reduced set → fewer rows); destructive rows styled; unsupported `block` never rendered.
- `test/widgets/chat/conversation_search_test.dart`: `searchLoaded` ordering; next/prev wrap; count label; highlight spans.
- `test/providers/prefs_chat_actions_gateway_test.dart`: mute until, pin, unread round-trip with `SharedPreferences.setMockInitialValues`.
- Goldens (brief §26 #11–14): personal chat normal, actions sheet open, group chat with chip, group info with orbit (3 participants + 2 observers).

---

## 8. Acceptance

- A group with an active Susu shows a quiet chip, never a banner; tapping it opens the dashboard.
- Group info clearly separates who is in the Susu from who is watching; observer count never inflates participants; no amount is shown to someone the server did not give it to.
- The three-dot menu contains only working actions; each one does what its label says, with scope-accurate copy.
- In-conversation search highlights matches and steps through them without pushing a new screen.
- Mute suppresses local notifications for that chat on this device and says so.