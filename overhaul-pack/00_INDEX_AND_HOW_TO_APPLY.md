# AZM Deep Experience Overhaul — Implementation Pack (Index)

**Repository:** `AzamanLTD/AZM-frontend` · **Written against:** the `AZM-frontend-main` snapshot (main `30188ef25b9d09456409e15a4a44f12ee9458ab5` at brief time)
**Predecessor documents:** `AZM_UI_Correction_Spec_v2.md` (phases A–D) · `AZM_UI_Correction_Agent_Handoff_v3.md` (currently being implemented) · `Azaman_Companions_Spec.md`

This pack is the code-level implementation of `AZM_Deep_Experience_Overhaul_Agent_Brief_v1.md`. It is written so that an implementation agent working on the live GitHub repo can apply it **surgically**: every section names the file it touches, quotes the anchor code that exists today, and gives the replacement or the new file in full.

---

## 0. Reading order

| # | Document | Brief sections it implements | Depends on |
|---|----------|------------------------------|------------|
| 00 | this file | §27–30 (order, PR discipline, reporting) | — |
| 01 | `01_FOUNDATION_EXPERIENCE_MODEL.md` | §3, §4, §20, §21, §22 — audit map, spatial modes, intents, motion contracts, gateways, test conventions | v3 Phase B (`AzNavState`) |
| 02 | `02_MARKETPLACE_DISCOVERY.md` | §5, §7, §11 — Marketplace home redesign, unified search state, merchant cards, world memory, resume, trust language | 01 |
| 03 | `03_MARKETPLACE_VERTICALS.md` | §6, §7.2 — Retail, Restaurants (shared menu model), Hotels, Transit journey thread, logistics seam | 01, 02, v3 Phase D |
| 04 | `04_CHAT_HUB_AND_STORY_RAIL.md` | §8, §14 (hub part) — Inbox redesign, collapsed-by-default pull-down story rail as a deterministic scroll-owned state machine | 01 |
| 05 | `05_STORY_VIEWER.md` | §9, §11 — multi-creator viewer, gesture matrix, media lifecycle, interaction layer, business action tray | 01, 04 |
| 06 | `06_STORY_CREATION_EDITOR_PRIVACY_ANALYTICS.md` | §10 — camera, media selector, scene/document editor model, `StoryEditSession`, privacy, lifespan, view-once, analytics | 01, 05 |
| 07 | `07_SUSU_IN_CHAT_GROUPS_AND_CHAT_ACTIONS.md` | §12, §13, §14 — Susu social proof, participants vs observers, member orbit, explainer, real three-dot menu, in-conversation search, group identity | 01, 04 |
| 08 | `08_COMPANION_ARCHITECTURE.md` | §23, §24 — additive companion lane with authenticity, dedupe, manifest, privacy, a11y | 01, 07 |

Phases A–D of the v3 handoff (Add Cash, typography, context-reactive nav, Home, store correction) are **not re-specified here**. Where this pack touches them it references the names the v3 handoff mandates — `AzNavState`, `NavTabs`, `NavPortal`, `PortalContext`, `storeNavContextProvider`, `StoreSheetGeometry`, `AzRadius.card` — and assumes they exist when the corresponding phase is applied.

---

## 1. Assumptions the agent must verify first

1. `main` has moved since `30188ef…`. Verify the anchor code quoted in each section still exists before editing (`git grep` the anchor). If an anchor is gone, stop, report, and re-locate before applying.
2. v3 Phase B (`lib/widgets/premium_bottom_nav.dart`, `lib/widgets/contextual_nav_band.dart`, `AzNavState`) has landed or is landing in an open PR. Document 02 §4 (unified search state) plugs into `PortalContext.onChanged/onSubmit`; if Phase B is not yet merged, implement the search state anyway (it is self-contained) and leave a `// TODO(nav-b)` seam where the pill wires in.
3. v3 Phase D (store correction) supersedes PR #135. Document 03 only adds *content* inside the corrected sheet; it does not touch the sheet geometry.
4. `Azaman_Companions_Spec.md` is reviewed-not-implemented in the v3 pass. Document 08 only adds the architecture lane and never touches money surfaces.

---

## 2. Suggested PR slicing (one phase per PR, keep ≤ 2 open)

| PR | Contents | Docs |
|----|----------|------|
| F1 — Foundation | `lib/experience/**` (spatial modes, intents, motion solvers, gateway interfaces, demo adapters), test conventions, architecture map doc in repo (`docs/ARCHITECTURE_MAP.md`) | 01 |
| M1 — Marketplace discovery | Marketplace home portal rebuild, `MarketplaceSearchState`, world memory, resume card, trust mark, merchant card | 02 |
| M2 — Verticals | Shared `MenuDocument`, retail quick-peek + tray, hotel stay bar, transit journey thread, service seam | 03 |
| C1 — Chat hub + story rail | `FriendsHubScreen` body → single `CustomScrollView`, `StoryRailHeaderDelegate`, `StoryRailSnapPhysics`, inbox rows | 04 |
| S1 — Story viewer | New `StoryViewerScreen` architecture | 05 |
| S2 — Story creation | Scene model, editor rebuild, privacy/lifespan, analytics | 06 |
| G1 — Susu + chat actions | Susu chip/rail/orbit/explainer, three-dot action sheet, conversation search | 07 |
| K1 — Companion lane | Models, manifest, gateway, reaction layer (no production surface wiring) | 08 |

Never merge to `main`. Report per the v3 §13 / brief §30 contract at the end of each PR (base SHA, head SHA, PR #, files, tests, `flutter analyze`, CI run id, on-device observations, deviations, next slice).

---

## 3. House rules that apply to every file in this pack

- Reuse tokens: `AzSpace`, `AzRadius`, `AzText`, `MotionTokens`, `AzMotion`, `AzSheetGeometry`, `AzamanHaptics`, `AzamanSheet.showPanel`. No new raw magic numbers unless the doc introduces a named token.
- Reduced motion is read through **`AzMotion.of(context).travel`** (which already folds the OS setting and the `AzSensory` override). Collapse, don't slow.
- One gesture, one owner. Every interaction section names the owner explicitly.
- No network call from a subtree rebuilt on scroll/animation ticks. Fetches live in providers/notifiers; widgets only `ref.watch`.
- No fake richness: every signal widget takes an explicit nullable input and renders nothing when the input is null.
- Reads of raw `Map<String,dynamic>` stop at the model boundary. New code introduces typed view models.
- Tests go through `test/goldens/golden_harness.dart`'s `pumpGoldenSurface` for visual states; behavior tests use `tester.pump(Duration)` on controllers, never `Future.delayed` sleeps.

---

## 4. New top-level directory introduced by this pack

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
  demo/
    demo_guard.dart
    demo_story_gateway.dart
    demo_chat_actions_gateway.dart
```

Everything else lands next to the code it extends (`lib/screens/friends/...`, `lib/widgets/stories/...`, `lib/widgets/susu/...`, `lib/widgets/marketplace/...`, `lib/widgets/companion/...`).