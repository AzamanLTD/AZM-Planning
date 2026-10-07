# AZM Deep Experience Overhaul — Implementation Pack (Index)

**Repository:** `AzamanLTD/AZM-frontend` · **Written against:** live `main` `6810007f5057334d0d2aab97a1294e861f208bca` (after PR #135 `MarketStorefrontShell` and PR #136 Add Cash + Comic Neue typography) · **Backend contracts:** `AzamanLTD/AZM-backend` `main` (`routes/storyRoutes.js`, `routes/storyHighlightRoutes.js`, `services/storyService.js`, `services/storyHighlightService.js`)
**Predecessor documents:** `AZM_UI_Correction_Spec_v2.md` (phases A–D) · `AZM_UI_Correction_Agent_Handoff_v3.md` (A and the storefront correction have landed as #136 / #135; B–C partially, see §1) · `Azaman_Companions_Spec.md` (reviewed, not implemented)

This pack is the code-level implementation of `AZM_Deep_Experience_Overhaul_Agent_Brief_v1.md`. It is written so that an implementation agent working on the live GitHub repo can apply it **surgically**: every section names the file it touches, quotes the anchor code that exists today (tagged **EXISTING** with `path:line`), and gives the replacement or the new file (tagged **NEW**) in full.

**Status of this pack:** reviewed UI-overhaul blueprint / future implementation backlog. It is **not** active execution authority. `ROADMAP.md` remains the single active execution plan and `CURRENT_STATE.md` still names the financial/checkout workstream as the active priority. Waves from this pack become active only when deliberately inserted into `ROADMAP.md` and `CURRENT_STATE.md` in their own planning PR; nothing in this pack edits those files.

---

## 0. Reading order

| # | Document | Brief sections it implements | Depends on |
|---|----------|------------------------------|------------|
| 00 | this file | §27–30 (order, PR discipline, reporting) | — |
| 01 | `01_FOUNDATION_EXPERIENCE_MODEL.md` | §3, §4, §20, §21, §22 — audit map, spatial modes, intents, motion contracts, gateways, test conventions | nothing new; builds on EXISTING `AzMotion`, `MotionTokens`, `AzSheetGeometry` |
| 02 | `02_MARKETPLACE_DISCOVERY.md` | §5, §7, §11 — Marketplace home redesign, unified search state, merchant cards, world memory, resume, trust language | 01 |
| 03 | `03_MARKETPLACE_VERTICALS.md` | §6, §7.2 — Retail, Restaurants (shared menu model), Hotels, Transit journey thread, logistics seam | 01, 02, EXISTING `MarketStorefrontShell` (#135) |
| 04 | `04_CHAT_HUB_AND_STORY_RAIL.md` | §8, §14 (hub part) — Inbox redesign, collapsed-by-default pull-down story rail as a deterministic scroll-owned state machine | 01 |
| 05 | `05_STORY_VIEWER.md` | §9, §11 — multi-creator viewer, gesture matrix, media lifecycle, interaction layer, business action tray | 01, 04 |
| 06 | `06_STORY_CREATION_EDITOR_PRIVACY_ANALYTICS.md` | §10 — camera, media selector, scene/document editor model, `StoryEditSession`, privacy, lifespan, view-once, analytics | 01, 05 |
| 07 | `07_SUSU_IN_CHAT_GROUPS_AND_CHAT_ACTIONS.md` | §12, §13, §14 — Susu social proof, participants vs observers, member orbit, explainer, real three-dot menu, in-conversation search, group identity | 01, 04 |
| 08 | `08_COMPANION_ARCHITECTURE.md` | §23, §24 — additive companion lane with authenticity, dedupe, manifest, privacy, a11y | 01, 07 |

---

## 1. What exists on live `main` today (verified, not assumed)

The v3 handoff names `AzNavState`, `NavTabs`, `NavPortal`, `PortalContext`, `storeNavContextProvider`, `StoreSheetGeometry` **do not exist** on live main and are **not referenced anywhere in this pack anymore**. The live foundation these docs build on is:

| Concern | EXISTING on live main |
|---|---|
| Shell ↔ nav bus | `AppShellBus` + singleton `appShellBus` — `lib/widgets/contextual_nav_band.dart:157/203` (`engage({onTabRequest, initialTab})`, `disengage()`, `setActiveTab(int)`, `requestTab(int)`; tabs Home `0` · Chat `1` · Marketplace `2`) |
| Contextual band above navigator | `ContextualNavBand` — `lib/widgets/contextual_nav_band.dart:209` (route→title map keyed by `AzRouteNames`) |
| Bottom pill | `PremiumBottomNav(trailing:)`, `navScrollCompression` (`ValueNotifier<double>`), `NavScrollCompression`, `TabScrollRegistry`, `NavRetapController` — `lib/widgets/premium_bottom_nav.dart:40–274`; `+` trigger `PlusLauncherTrigger` (`lib/main.dart` shell) placed with `solvePanel()` from `lib/widgets/liquid/liquid_placement.dart` |
| Store page | `MarketStorefrontShell` (`lib/screens/marketplace/market_storefront_shell.dart:68`; ctor: `business, colors, bannerUrl, locations, showcaseSlides, isFollowing, unpaidInvoices, hasCatalog, catalogLabel, productsBuilder, onBack, onToggleFollow, onShare, onPrimaryCta, onOrderProduct, onOpenCatalog, onOpenReviews, onOpenLocations, onPayInvoice, onLaunch`) + `MarketStorefrontSnaps {info .46, overview .62, shopping .78}` (`:55`); hosted by `lib/screens/marketplace/business_profile_screen.dart:1172`. `collapsible_business_bar.dart` **no longer exists**. |
| Routing | `AzRouteNames` (names, `lib/router/route_registry.dart:24`) **and** `AzRoutes` (paths, `:128`). Both exist. Navigation uses paths: `context.push(AzRoutes.x)` (55 call sites) / `context.go` (9). This pack uses `AzRoutes` for paths and `AzRouteNames` only where a name is needed (band titles). Builders used here: `AzRoutes.businessProfile(bizId)` → `/business/$bizId`, `AzRoutes.savedBusinesses` → `/biz/saved`, `AzRoutes.susuDetail(id)`, `AzRoutes.businessStories(bizId)`. There is **no** `AzRoutes.cart` — see 02 §7.7. |
| Typography | `AzText.uiFamily = 'ComicNeue'`, `AzText.numericFamily = 'Inter'` (`lib/theme/az_text.dart:79–80`); Comic Neue is bundled. Money surfaces never use the UI family. |
| Radii | `AzRadius {xs 4, sm 8, md 12, lg 16, xl 20, xxl 28, pill 999}` + `brXs…brPill`, `sheetTop`, `sheetTopLg`, `topLg` (`lib/theme/az_radius.dart`). There is **no** `AzRadius.card`; cards use `AzRadius.lg`. |
| Business hours | `BusinessLocation.operatingHours` (`lib/models/business_models.dart:636`, `{mon: "8:00-22:00"}`), parsed today only in `lib/widgets/business_card.dart:374 _isOpenNow()`. `businessMeta` carries `showcaseUrls`, **not** hours. |
| Stories (frontend) | `lib/screens/story_viewer_screen.dart` (`StoryViewerScreen.open` `:25`), `lib/screens/story_editor_screen.dart`, `lib/screens/story_camera_screen.dart`, `lib/screens/story_creation_screen.dart` (`_uploadStory` `:32`, `apiClient.multipart('/stories', …)` `:51`), `lib/widgets/story_ring.dart`, `lib/models/story_model.dart` (`StoryItem{id, mediaUrl, mediaType, caption, linkedBizId, durationSeconds, boosted, seen, createdAt}` — **no `expiresAt`**), `lib/providers/story_provider.dart` (`/stories/feed`, `/stories/$id/view|boost|reply`). |
| Stories (backend) | Mounted at `/api/stories` (`src/routes/index.js:56–57`). `POST /` multipart `file` + `caption`, `linkedBizId`, `durationSeconds` → `storyService.create` **hardcodes `expiresAt = now + 24h`**; `GET /feed` returns per story only `{id, mediaUrl, caption, linkedBizId, boosted, seen, createdAt}` (no `mediaType`, `durationSeconds`, `expiresAt`); `POST /:id/view`, `POST /:id/boost`, `DELETE /:id`; highlights `GET/POST /highlights…`; `GET/POST/DELETE /close-friends…`; `GET /analytics/:storyId`, `GET /analytics/business/:businessId` (`{stories: [{viewCount, uniqueViewerCount, reactionCount, replyCount, shareCount, profileClickCount, story{id, mediaUrl, caption, createdAt, expiresAt}}], totals: {totalViews, totalUniqueViewers, totalReactions, totalReplies, totalShares, totalProfileClicks}}`). **No** `privacy`, `expiresInSeconds`, `viewOnce`, `sceneJson`, `PATCH /:id`, `/reply` in routes (frontend posts to `/stories/$id/reply`; verify server before relying on it). |
| Notifications | Visible in-app banner = `InAppPushBanner.show(ctx, title:, body:, onTap:)` (`lib/widgets/in_app_push_banner.dart:26`), called from `lib/main.dart:745 _showSocketNotificationBanner` on socket `new_notification` / `new_trade_request`; FCM foreground = `push_notification_service.dart:68 → _handleForeground`. Socket `friend_message` is consumed by `friend_provider.dart:65 _handleFriendMessage` (unread bump) and `premium_chat_provider.dart`; it never shows a banner today. `flutter_local_notifications` is in `pubspec.yaml` but `FlutterLocalNotificationsPlugin` is not referenced in `lib/`. |
| Susu | `SusuFrequency {daily, weekly, biweekly, monthly, unknown}` (`lib/models/susu_model.dart:120`); GHS rate = `SusuSuppliedRate{usdcToGhs, source}` via `susuSuppliedRateProvider` (`lib/providers/susu_provider.dart:501–532`). Group anchors: `lib/screens/group_chat/group_chat_screen.dart` `_SusuEventCard :409`, `_SusuBanner :436`; `group_profile_screen.dart` `_InitiateCta :386`, `_ActiveSusuBadge :576`. |
| Chat | `lib/screens/friends/friend_chat_screen.dart` three-dot `Icons.more_vert → _openChatProfile` `:605–608` (`_openChatProfile :476`, `_openSearch :285`); hub `lib/screens/friends/friends_hub_screen.dart` `'Inbox' :336`, `_pickAndCreateStory :69/:474`, `StoryViewerScreen.open :542`, `_buildChatsTab :857`, `_ChatListEntry :960`. |
| Realtime | `RealtimeEventDeduper({maxEntries = 256}).accept(id)` (`lib/services/realtime_event_deduper.dart:13`); `SocketService` listener-list pattern (`onNewNotification`, `_newNotificationListeners`, `socket_service.dart:67/279`). |
| CI | `.github/workflows/flutter-ci.yml`: Flutter `3.47.5` pinned, `flutter analyze --no-fatal-infos --no-fatal-warnings`, `flutter test --coverage`. Goldens via `test/goldens/golden_harness.dart` `pumpGoldenSurface`. |
| Media deps | `image_picker`, `video_player ^2.9.2`, `firebase_messaging`, `shared_preferences`; **no** `camera` package. |

---

## 2. Assumptions the agent must verify first

1. `main` moves. Before editing, `git grep` every **EXISTING** anchor quoted in the section being applied. If an anchor is gone, stop, report, and re-locate before applying — never recreate a deleted anchor to make the doc fit.
2. The search pill in the nav (brief §5.4) is **not** on live main. Document 02 §4 implements the search state so that it is the only owner of marketplace search text today (bound to the existing explore `TextField`) and exposes a `MarketplaceSearchBinding` seam for the pill when it lands. Do not block 02 on nav work.
3. The store page is `MarketStorefrontShell` (#135). Document 03 only adds content through the shell's `productsBuilder`/callbacks and shared models; it does not touch the sheet geometry (`MarketStorefrontSnaps`).
4. `Azaman_Companions_Spec.md` is reviewed-not-implemented. Document 08 adds the architecture lane only and never touches money surfaces.
5. Backend story capabilities (privacy, lifespan, view-once, edit, scene) are **absent** today. Document 06 ships them gated and reports `StoryGateway.capabilities == {}` for all of them until a backend PR exists. Nothing in the editor may produce a result the server would drop.

---

## 3. PR slicing (one document per PR; complete, reviewable slices)

| PR | Contents | Docs |
|----|----------|------|
| F1 — Foundation | `lib/experience/**` (spatial modes, intents, motion solvers, gateway interfaces, demo adapters), test conventions, architecture map doc in repo (`docs/ARCHITECTURE_MAP.md`) | 01 |
| M1 — Marketplace discovery | Marketplace home portal rebuild, `MarketplaceSearchState`, `BusinessHours`, world memory, resume card, trust mark, merchant card | 02 |
| M2 — Verticals | Shared `MenuDocument`, retail quick-peek + tray, hotel stay bar, transit journey thread, service seam | 03 |
| C1 — Chat hub + story rail | `FriendsHubScreen` body → single `CustomScrollView`, `StoryRailHeaderDelegate`, `StoryRailSnapPhysics`, inbox rows | 04 |
| S1 — Story viewer | New `StoryViewerScreen` architecture | 05 |
| S2 — Story creation | Scene model, editor rebuild, gated privacy/lifespan, analytics | 06 |
| G1 — Susu + chat actions | Susu chip/rail/orbit/explainer, three-dot action sheet, conversation search | 07 |
| K1 — Companion lane | Models, manifest, gateway, reaction layer (no production surface wiring) | 08 |

Rules:
- **Never push to `main`.** Every slice is a branch + PR; merge only after review and green exact-head CI.
- **No changed-line budget.** A slice is sized by coherence, not line count (planning `ACTIVE_LOOP.md` operating contract #8). Split when the scope is logically too large, never to satisfy a metric. Every implementation slice carries executable tests; planning-document slices carry none and say so.
- Report per the brief §30 contract at the end of each PR: base SHA, head SHA, PR #, files, tests, `flutter analyze`, CI run id, on-device observations, deviations, next slice.

---

## 4. House rules that apply to every file in this pack

- Reuse tokens: `AzSpace`, `AzRadius`, `AzText`, `MotionTokens`, `AzMotion`, `AzSheetGeometry`, `AzamanHaptics`, `AzamanSheet.showPanel`. No new raw magic numbers unless the doc introduces a named token.
- Reduced motion is read through **`AzMotion.of(context).travel`** (which already folds the OS setting and the `AzSensory` override). Collapse, don't slow.
- One gesture, one owner. Every interaction section names the owner explicitly.
- No network call from a subtree rebuilt on scroll/animation ticks. Fetches live in providers/notifiers; widgets only `ref.watch`.
- No fake richness: every signal widget takes an explicit nullable input and renders nothing when the input is null.
- **Raw `Map<String, dynamic>` stops at the model boundary.** New code introduces typed view models; no proposed widget reads `businessMeta[...]`, `params[...]` or analytics JSON directly (02 §3, 06 §3, 06 §8).
- **Capability truth.** Every gateway reports what the server does today; UI controls for unsupported capabilities are absent, not disabled-with-a-tooltip.
- Tests go through `test/goldens/golden_harness.dart`'s `pumpGoldenSurface` for visual states; behavior tests use `tester.pump(Duration)` on controllers, never `Future.delayed` sleeps.

---

## 5. New top-level directory introduced by this pack

```
lib/experience/
  az_spatial_mode.dart            // spatial modes vocabulary
  az_intent.dart                  // interaction intents
  az_surface.dart                 // surface taxonomy
  motion/
    az_pull_reveal_controller.dart
    az_snap_solver.dart
    az_identity_morph.dart
  gateways/
    az_gateway_result.dart
    story_gateway.dart
    chat_actions_gateway.dart
    marketplace_discovery_gateway.dart
    susu_membership_gateway.dart
    service_flow_gateway.dart
    room_membership_resolver.dart   // 08: real authorization seam (no `=> true`)
  demo/
    demo_guard.dart
    demo_story_gateway.dart
    demo_chat_actions_gateway.dart
```

---

## 6. Review reconciliation (PR #60 review → where each point is resolved)

| # | Review point | Resolved in |
|---|---|---|
| 1 | Anchors drifted from `30188ef`; `AzNavState`, `MarketplaceSearchScope`, `storeNavContextProvider`, `StoreSheetGeometry`, `BusinessHours` not on main | 00 §1; every doc's EXISTING/NEW tags; 02 §3 defines `BusinessHours` as NEW against `BusinessLocation.operatingHours` |
| 2 | Invented API/router names | `AzRoutes` is real (paths) and kept; `AzRoutes.business(...)` → `AzRoutes.businessProfile(...)`; `AzRadius.card` → `AzRadius.lg`; v3-only nav names removed everywhere; store scope hand-off re-anchored on `business_profile_screen.dart` (02 §9) |
| 3 | Backend capability boundaries explicit (stories) | 06 §5.2 (upload sends only fields the server reads), §5.3 (edit unsupported), §6–7 (privacy/lifespan/view-once gated, capabilities `{}` today), §10 acceptance; 05 §2 (feed field gaps) |
| 4 | StoryEditor history bug (`take(_maxHistory)` keeps oldest) | 06 §3.2 keeps the **most recent** 50 + test |
| 5 | Raw-map leaks (`businessMeta['hours']`, `WidgetIntentLayer.params`) | 02 §3 typed `BusinessHours`/`OpenState` from typed locations; 06 §3 sealed `WidgetIntent` payloads |
| 6 | Notification anchor pointed at `push_notification_service.dart` | 07 §5.1 re-anchored on `main.dart:745 _showSocketNotificationBanner` / `InAppPushBanner.show`, `_handleForeground`, `friend_provider.dart:65` |
| 7 | Susu "every week" hardcoded; GHS conversion implicit | 07 §3.2 cadence from `SusuFrequency`; `SusuSocialView.contributionGhs` + `rateSource` from `susuSuppliedRateProvider` |
| 8 | Companion `_isMemberOf(...) => true` | 08 §3 `RoomMembershipResolver` seam with provider-backed impl + tests |
| 9 | Governance / slicing | 00 status paragraph; one doc per planning PR; implementation one slice per PR; no line-count target |
| 10 | Analytics model names drifted (`uniqueViewers`, `reactions`) | 06 §8 typed model derived field-for-field from `storyHighlightService` response |
| 11 | Video overlays flattened claim | 06 §5.2 / §10: image-only flattening; video overlay tools absent until a backend `sceneJson` contract exists |
