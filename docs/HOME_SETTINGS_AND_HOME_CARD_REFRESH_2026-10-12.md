# HOME, SETTINGS AND HOME-CARD REFRESH — Implementation Specification

**Date:** 2026-10-12
**Status:** PROPOSED — under review (Planning PR)
**Contract source:** founder implementation contract dated 2026-10-12 (Home header, profile drawer, themes, continuing experience programme)
**Repos:** `AzamanLTD/AZM-frontend` (implementation), `AzamanLTD/AZM-Planning` (this document)
**Frontend main at audit:** `9640face067f45d0cfda6bb95f1569af6878add7` — VERIFIED live on 2026-10-12 (`git rev-parse origin/main`); matches the founder's reference SHA. Main has NOT advanced past it.
**Working tree at audit:** clean on `feat/receive-request-recent-interaction` (the PR #147 branch; no uncommitted changes).

---

## 0. Scope and binding direction

The founder's specified behaviour is binding. This document records it durably so no part of it depends on a single chat transcript. Two classes of work:

1. **New visual/interaction work** (Home header, physical card stack, logo-fill launcher, settings drawer profile hero + QR, drawer decluttering, Themes/accent palette, Settings page hierarchy).
2. **Continuing programmes** that must NOT be displaced or duplicated: persistent-navigation occlusion, single-contact Request composer (PR #147 follow-up), measured route-entry performance, Marketplace Mix (owned by the overhaul-pack programme, see §7.4), and the reactive balance-card effect (§7.7).

Do not reduce the assignment to a styling pass. Preserve real functionality, account security, navigation semantics, accessibility, performance and existing design-system decisions. No pushes to `main`; every change goes through a PR left open and unmerged. Green CI does not authorize merging.

## 1. Verified code-path audit (2026-10-12)

All EXISTING claims below were verified by reading the live files at main `9640fac` on 2026-10-12. Anything not verified is labelled UNVERIFIED.

### 1.1 Home header (`lib/screens/home_screen.dart`, 1133 lines)

- **EXISTING** — `_GreetingHeader` (line 939) composes, left→right: initials avatar (44px `ScaleTap` + `Hero('profile-avatar')`, pushes `ProfileScreen`), `Spacer`, AZM Rewards pill (`PremiumGlassContainer`, `HugeIconsSolid.gift`, "N AZM" / "Earn AZM", pushes `AzmRewardsScreen`), `NotificationBell`, balance-visibility eye (`ScaleTap` + `PremiumGlassContainer` 40×40 + `AnimatedSwitcher` rotating `HugeIconsStroke.view`/`viewOff`, toggles `balanceVisibleProvider`, `AzamanHaptics.toggle()`).
- **MODIFIED (PR-A)** — notifications move to the far-left (former avatar position); profile button to the far right, opening the existing end drawer via an explicit callback (see §2.1); Rewards pill and Home entry removed; eye relocated to the physical balance card (§2.2).
- **EXISTING** — shell ownership: `lib/main.dart` owns the `Scaffold` with `GlobalKey<ScaffoldState> _scaffoldKey` (line 502) and `endDrawer: const SettingsDrawer()` (line 830). `AzamanHomePage` is injected as `const AzamanHomePage()` in `_pages` (line 523) and `_pageFor` (line 561). Adding a constructor callback breaks `const` at both call sites; test hosts (`test/widgets/home_*.dart`, 7 files) construct `AzamanHomePage()` inside their own `Scaffold` without a drawer and must keep working with a test callback.

### 1.2 Physical card stack

- **EXISTING** — `lib/widgets/home/pull_reveal_card_deck.dart` (249 lines): `PullRevealPhysics` owns drag→progress→reveal mapping with ease-in resistance, commit threshold + threshold haptic, spring settle (stiffness 380 / damping 26), programmatic collapse hook for the PIN gate. `_cardHeight = 180` host constant confirmed in the deck's host (verified in the Home build; exact constant lives in the deck host composition).
- **EXISTING** — PIN gate (`home_pin_gate_test.dart`, `home_card_deck_test.dart`): protects expanded card details.
- **EXISTING** — `lib/widgets/flippable_balance_card.dart` (568 lines) + `lib/widgets/hologram_balance_card.dart` (375 lines): cool near-black/graphite material; USDC principal + GHS secondary; lighter main figure; live rate + refresh countdown; hidden-state eye indicator on the card face (to be replaced by the relocated full control).
- **EXISTING** — `lib/widgets/home/azm_visa_card.dart` (517 lines): lower Visa-concept card.
- **MODIFIED (PR-A)** — footprint reduction ~8–12% (evaluate 166–165px against real layout constraints, responsive, no text clipping); depth treatment via surroundings (contact shadow, restrained rim light, differentiated backing surface — NOT another stacked border); curved cradle/stage silhouette around the lower-card housing; improved handoff (balance card recedes via small scale/depth change, lower card advances as next front card) driven from the deck's existing state; two subtle position dots bound to real deck state with a11y semantics; eye integrated into `HologramBalanceCard` header aligned with AVAILABLE, no duplicate indicators, and eye taps must NOT flip the card (nested-gesture audit required).
- **PRESERVE** — `HomeReminderDeck` ordering/shuffle/first-card/next-card peek/dots, Recent Activity scroll handoff (resisted pull, snap, doorway) and the doorway-over-nav geometry (`home_doorway_geometry_stability_test.dart`). If card-size changes affect measured geometry, adjust the affected measurements deliberately.

### 1.3 Action launcher (`lib/widgets/home/plus_action_launcher.dart`, 607 lines)

- **EXISTING** — `PlusLauncherController` (open/close/toggle, `fromTriggerRect`), anchored trigger beside the nav pill, rotation animation (`AnimationController _rot`, `Transform.rotate`, plus→×), overlay tied to measured global rect, existing action set.
- **EXISTING** — logo-fill vocabulary: `lib/widgets/azaman_send_button.dart` (257 lines) — 3-part AZM logo (`_path1.._path3`), progressive per-part fill (biggest→middle→smallest, on completion "sends"; reverse unfill smallest→middle→biggest), `_AzamanSendPainter`.
- **MODIFIED (PR-A)** — launcher displays the real AZM logo; tap plays a CUMULATIVE bottom-up fill (lower segments stay filled; unlike the chat button's sequential empty-then-fill); menu opens only after the final segment completes; rotation removed entirely (no ×); close/reset resets visual state predictably; one well-owned animation controller, interruption/rapid-tap safe; reduced motion preserves semantics/final state while skipping travel; all existing actions, routing, anchor measurement and semantics preserved.

### 1.4 Settings drawer (`lib/widgets/settings_drawer.dart`, 1191 lines)

- **EXISTING** — persistent `Scaffold.endDrawer` (shell line 830), live user state, width `MediaQuery.sizeOf(context).width * 0.88` (line 67), `HQ SHORTCUTS` section (line 207; Deposit/Withdraw/History/Leaderboard), `Vaults` button (line 254), Rewards expansion (lines 863+, AZM Auction + AZM Rewards + Referral), payment-addresses, recommendations, security and business entry points.
- **MODIFIED (PR-B)** — profile hero at drawer top: real profile picture horizontally centred relative to the DRAWER's width, protruding from the hero surface, @username below, real scannable QR preview beside the identity row; hero tap opens the genuine profile edit flow (audit `SettingsScreen` → `ProfileDetailsScreen` first; NOT `ProfileScreen`); QR tap opens a larger scan-ready popup with the centred `assets/images/azaman_logo.png` mark (verified asset exists) over a light backing, using `qr_flutter: ^4.1.0` (verified in pubspec). QR payload contract: `azaman://user/$username`, preserved end-to-end with `lib/screens/qr_scanner_screen.dart` (recognises `azaman://user/{username}` → `/friends/search?q=` → username match → friend request). UNVERIFIED in this audit pass: the exact ProfileDetailsScreen edit-capability set (field coverage) — verify before wiring; document any gap instead of directing users to an unrelated screen.
- **MODIFIED (PR-B)** — Telegram-inspired scroll-driven avatar: pinned sliver header with custom delegate, avatar sinks toward the header as content scrolls, dimensions from the drawer's OWN constraints + `Media MediaQuery.padding`/display features, no hardcoded notch, works on flat-top Android and notchless iOS; animation local to the header layer (no whole-drawer rebuilds per tick), no permanent idle loop, reduced-motion respected.
- **MODIFIED (PR-B)** — width-centring: all positioning derived from drawer-local constraints (works in landscape and different widths); review the SafeArea/backdrop-blur cost vs clarity, choose the cheapest convincing treatment.
- **REMOVED (PR-B)** — `HQ SHORTCUTS` heading/grid, `Vaults` drawer button, AZM Rewards entry points from Home and drawer (do NOT relocate the gamified Rewards surface back into the drawer; do NOT fabricate leaderboards/badges/counters; underlying Deposit/Withdraw/Vault/History/Referral/KYC/security/auction features stay available at canonical routes; reference-audit imports before deleting).
- **RECORDED, NOT IMPLEMENTED** — Azaman trust/credit score: no authoritative backend model exists to display; do not invent. Record for separate product decision.
- **MODIFIED (PR-B)** — regroup remaining drawer destinations: Profile & identity / Money & payment methods / Security & privacy / Appearance & preferences / Business tools (contextual, role-valid only). One coherent icon vocabulary chosen from installed sets (`hugeicons` package + Material audit — verify installed glyph constants before referencing).

### 1.5 Themes, accents, Settings page

- **EXISTING** — `enum AzAccent { gold, teal, indigo, rose }` in `lib/providers/theme_provider.dart` line 42; persisted integer indices under `azaman_accent` in SharedPreferences; `AzAccentFamily` light/dark accent/secondary/on-accent definitions. UNVERIFIED this pass: exact `AzAccentFamily.all` composition and the `azaman_accent` load/save call sites — verify before editing (index-compatibility is critical).
- **EXISTING** — `AccentIdentityRow` embedded in `lib/widgets/sensory_preferences_section.dart` (355 lines); Themes screen `lib/screens/theme_picker_screen.dart` (283 lines) primarily light/dark preview cards.
- **MODIFIED (PR-C)** — move `AccentIdentityRow` to the Themes experience; sensory preferences (haptics/sound/ambient/reduced-motion) stay in sensory settings. Extend `AzAccent` to ~8 identities by APPENDING only (candidate additions: blue, violet, emerald, coral — evaluate against the surface ladder, dark-first, light-usable); never reorder/renumber existing entries (persisted indices). Complete `AzAccentFamily.all` per addition, readable on-accent contrast in both themes, labels + accessible selected states. Themes page: light/dark selection without changing the intended dark default, live accent preview across real surfaces, active state immediately visible, apply without restart/network, persistence across restarts.
- **MODIFIED (PR-C)** — `lib/screens/settings_screen.dart` (783 lines): full inventory before reorganising (profile edit via `ProfileDetailsScreen`, Theme, KYC, Security, password, account activity, saved wallet/MoMo destinations, referral, sensory, deactivation, sign-out, language/currency). Clear hierarchy per the contract's IA; every row leads to a real supported action; remove duplicates only with a verified canonical destination; coherent icon system; no wall of boxes. Accent controls live in Themes only.

### 1.6 Request / Receive (continuing programme)

- **VERIFIED** — frontend PR #147 open: `feat/receive-request-recent-interaction` @ `d3713dc` (base main), updated 2026-10-11, owned by this agent's earlier session. Do NOT take over underfoot; contract work continues on a follow-up branch or a coordinated update.
- **MODIFIED (PR-D)** — Request composer: replace the large contact list under the dialer with a compact **Select contact** row modelled on the Add Cash select-wallet interaction (verify that pattern in the deposit surfaces); opens a searchable bottom sheet; single contact per direct request; account ID (never friendship-row ID); LINK mode distinct. No multi-recipient, no keypad shrink, no legacy in-chat transfer repurposing, no fabricated rates/PAID status. Backend standalone payment-request contract (PR #332, MERGED to backend main `1cb18d3`, 2026-10-11) has NO pay/settlement flow; deployed runtime DB lacks `PaymentRequest`/`FinancialOperation` tables (create → 503 fail-closed, list → 500; re-trigger Render deploy is the unblock) — keep documented; never fake end-to-end success.

### 1.7 Other continuing programmes

- Persistent-nav occlusion (§7.1), route-entry performance (§7.3), Marketplace Mix (§7.4), public business pages (§7.5), pricing/availability truth (§7.6), reactive balance-card effect (§7.7) — see backlog inventory.

## 2. Design decisions (committed) and rationale

1. **Callback, not context-crawl:** Home's profile button receives an `onOpenSettings` callback injected by the shell (`AzamanHomePage(onOpenSettings: () => _scaffoldKey.currentState?.openEndDrawer())`). The shell owns the Scaffold; Home never reaches through a global context. Both `const AzamanHomePage()` call sites lose `const`; test hosts pass a recording callback. Trade-off: constructor churn at 2 app sites + 7 test hosts, accepted for explicit ownership.
2. **Eye relocation, not re-approximation:** the exact existing eye `ScaleTap`/`PremiumGlassContainer`/`AnimatedSwitcher` subtree moves into the `HologramBalanceCard` header next to AVAILABLE. The current hidden-state-only indicator on the card is removed, leaving one control. Nested-gesture conflict (eye tap flipping the card) is fixed in the gesture layers (absorb the tap in the eye target without disabling the card flip), then regression-tested — not worked around by disabling the flip.
3. **Physical-card depth:** one convincing lighting/material treatment (contact shadow + restrained rim light + differentiated backing surface) from existing tokens; no stacked bezels. Cradle silhouette is stage geometry around the lower card, never a mask across the Visa card face.
4. **Logo launcher:** reuse the send button's path data/painter vocabulary with a cumulative per-part fill (bottom-up), a single owned `AnimationController`, menu open chained after the final segment completes (with mounted check), reset on close. Reduced motion: skip travel, open menu on tap.
5. **Drawer hero:** `LayoutBuilder` + `SliverPersistentHeader` with a pinned delegate; the avatar morph is driven by the header's own minExtent/maxExtent interpolation (no extra scroll listener rebuilding the whole drawer). QR: real `QrImageView` (qr_flutter) at both sizes; error correction verified compatible with the centred logo backing.
6. **Accents:** append-only enum growth; persistence stays index-based so existing users keep their accent. New families derived through the existing `AzAccentFamily` architecture — no new theme implementations.
7. **PR separation:** six workstreams (§3) so review surfaces stay coherent; no giant mixed PR.

## 3. Workstream / PR ownership plan

| PR | Branch (frontend unless noted) | Contents | Status |
|----|----|----|----|
| Plan | `docs/home-settings-and-home-card-refresh-2026-10-12` (AZM-Planning) | This specification | opened 2026-10-12 |
| PR-A | `feat/home-controls-cards-logo-launcher` | Home header reorder + drawer callback, eye relocation + gesture fix, card footprint/depth/cradle/handoff + dots, logo-fill launcher | BLOCKED on start |
| PR-B | `feat/settings-drawer-profile-hero-qr` | Drawer profile hero, scrolling avatar, QR popup, declutter (HQ/Vaults/Rewards), regrouping | BLOCKED on start |
| PR-C | `feat/themes-accents-settings-ia` | AccentIdentityRow move, 8-accent palette, Themes page showcase, Settings hierarchy | BLOCKED on start |
| PR-D | `feat/receive-request-contact-picker` (follow-up to #147, based on verified #147 head or main+#147-merged, decided at start) | Select-contact row + searchable sheet, single-contact contract; plus persistent-nav occlusion fix if file ownership allows, else separate PR | BLOCKED on start |
| PR-E | `perf/route-entry-latency` | Measured route-entry performance (baseline + after timings or stated limitation + reproducible procedure) | BLOCKED on start |
| PR-F | owned by overhaul-pack programme (Planning pack/02, pack/03 + frontend overhaul branches) | Marketplace Mix | EXISTING programme, do not duplicate |

File ownership: PR-A owns `home_screen.dart` header block, `flippable_balance_card.dart`, `hologram_balance_card.dart`, `pull_reveal_card_deck.dart`, `azm_visa_card.dart`, `plus_action_launcher.dart`, `azaman_send_button.dart` (shared vocabulary, read-mostly), `main.dart` (2 call sites). PR-B owns `settings_drawer.dart` (+ new header/QR widgets), touches `home_screen.dart` only for the drawer callback already in PR-A. PR-C owns `theme_provider.dart`, `theme_picker_screen.dart`, `sensory_preferences_section.dart`, `settings_screen.dart`. PR-D owns `receive_screen.dart` (+ request composer widgets on its branch). Conflicts between PR-B/PR-C and PR-A at `home_screen.dart` are limited to the single profile-button row and are sequenced (PR-A first).

## 4. Unresolved product questions (for founder/review)

1. Final physical-card height after responsive evaluation (target ~165–166px; verify no clipping at 320 width + enlarged text before fixing).
2. Exact accent additions beyond gold/teal/indigo/rose — candidates blue/violet/emerald/coral, pending surface-ladder evaluation.
3. Whether the drawer's Profile hero should link to `ProfileDetailsScreen` (edit) vs a new dedicated edit route — decision after the ProfileDetailsScreen capability audit. Any missing field (e.g. photo upload) is documented, not invented.
4. Trust/credit score: intentionally not implemented (no authoritative model). Awaiting product decision + backend model, policy and evidence.
5. PR-D base: #147 is open and unmerged; follow-up branches from `feat/receive-request-recent-interaction` head (verified at start) so the contact-picker composes with the existing composer work.

## 5. Acceptance criteria and required tests (binding, from contract §12)

- **Header:** notifications at former profile position (far left); profile far right; profile opens the drawer (not `ProfileScreen`); Rewards pill + Home entry gone; no collision at 320 width / enlarged text.
- **Eye:** fully functional on the card; does NOT flip the card; hidden balances stay hidden front/back and across refresh/rebuilds; a11y preserved; same state on every surface.
- **Cards:** both fit, no overflow; reveal commits at existing threshold, settles, reverses, PIN gate respected (incl. cancel/fail collapse); dots match real state, accessible; Recent Activity handoff + reminder deck unchanged; stage silhouette reviewed via golden/screenshot fixture at mobile dimensions.
- **Launcher:** cumulative bottom-up fill, no rotation, menu opens only after final segment, close/reset predictable, rapid taps safe, all existing actions preserved, reduced-motion semantics intact.
- **Drawer/QR:** real user/photo data + placeholder/error fallback; centring within drawer width (incl. landscape); smooth expanded→collapsed motion (reverse scroll, fast fling, no-notch); hero opens genuine edit route; QR thumbnail opens modal, dismiss returns to same drawer state; popup QR encodes `azaman://user/$username`, decodes with centred logo, compatible with the add-friend scanner flow; no fabricated/foreign identity; HQ shortcuts/Vaults/Rewards removed while canonical routes remain reachable.
- **Themes/Settings:** saved theme/accent preserved; new accents persist; old indices compatible; readable contrast light+dark for every colour; security/profile-edit/currency-language/payment-destination/sign-out paths functional.
- **Matrix:** ~320/360/390/430 widths, small heights, enlarged text, dark+light, each accent, reduced-motion on/off, fast/reversed scroll, with/without top cutout.
- **Gates:** `dart format` on changed files, focused widget tests, `flutter analyze`, full Flutter Quality suite, Android Integration on the EXACT final head of each PR. Never loosen assertions to get green. Flaky environmental failures reported separately with evidence.

## 6. Delivery discipline

- Verify live SHAs/branches/PRs before each PR start (references in this doc are as of 2026-10-12).
- Never push to `main`; PRs stay open/unmerged for founder review; no Render/production/backend changes.
- While a PR's CI runs, start the next independent branch (avoid shared-file conflicts per §3).
- Report per PR: URL, branch, base, exact final HEAD SHA, changed files, summary, decisions/trade-offs, test totals, CI links tied to SHA, screenshots/goldens (drawer morph, cards, QR popup, launcher), measured performance or explicit statement of profiling limitation, blockers/dependencies, adjacent bugs + where recorded.

## 7. Pre-existing backlog inventory (must stay visible)

1. **Persistent bottom-navigation occlusion** — real layout defect: last items hidden behind the persistent bar on numerous screens. Fix: one consistent layout/inset abstraction aware of the persistent nav; regression tests for long lists, pushed screens, forms, short devices, enlarged text, keyboard-open. Keep nav timing/compression/interaction semantics. Owner: PR-D lane (or its own PR if conflicts).
2. **Single-contact Request composer** — see §1.6 / PR-D.
3. **Route-entry performance** — real reported delays (Home → Marketplace etc.). Distinct workstream (PR-E) with measured baselines; evidenced causes: pre-first-frame blocking, duplicate provider work, wide subscriptions, eager mounts. Never substitute animation-duration cuts for real work.
4. **Marketplace Mix** — owned by the overhaul-pack programme: Planning PRs #63 (`pack/02-marketplace-discovery`, `overhaul-pack/02_MARKETPLACE_DISCOVERY.md`, 899 lines) and #64 (`pack/03-marketplace-verticals`), plus frontend branches `feat/overhaul-02-marketplace-discovery`, `feat/overhaul-03-marketplace-verticals` (no open frontend PRs as of 2026-10-12 — ownership unverified, do not duplicate or clobber). Binding constraints: Eat/Shop/Ride/Stay + Mix mode; Stores merchant-first; Eat→Mix flagship; real products/authoritative prices/availability; no star-rating-driven discovery; no cross-business cart without settlement contracts; public business pages need verified web/SEO architecture; NEVER infer GHS price from `priceUsdc`/`priceRange`/FX; never imply a GHS checkout rail.
5. **Public business pages** — see 4; separate architecture, not a Flutter route bolt-on.
6. **Pricing/availability truth** — see 4's financial constraint; GHS authoritative price contract must be inspected (backend PRs) before discovery work.
7. **Reactive balance-card surface** — direction preserved: reuse existing `Listener` + normalized pointer position (no stolen horizontal gestures), restrained custom-painted radial glows between material and lighting layers, no unnecessary blur or perpetual idle animation, reduced-motion respected, graphite surface stays clean. Audit `lib/widgets/holographic_surface.dart` before implementing (EXISTING: `holographic_surface_test.dart`, `holographic_surface_particles_test.dart`). Record implemented vs remaining delta here when PR-A lands.

## 8. Reference ledger (verified 2026-10-12 unless noted)

- Frontend main: `9640fac` (verified). Reference SHA from contract matches.
- Frontend open PRs: only #147 (`feat/receive-request-recent-interaction` @ `d3713dc`, updated 2026-10-11).
- `feature/home-tickets-story-rail-savings`: 0 commits ahead of main, 3 behind — contains NO unmerged unique work; not a blocker; do not force-push or delete it (not ours to clean in this task).
- Planning main: `e14fe16` (verified at clone, 2026-10-12). Open Planning PRs: #60–#70 (overhaul pack + control-plane), do not touch.
- Backend payment-request contract: AZM-backend PR #332 MERGED at `1cb18d3` (2026-10-11, verified in earlier session). Deployed backend `azm-backend-9o0b.onrender.com` runs #332 code but its runtime DB lacks the new tables (create → `503 IDEMPOTENCY_UNAVAILABLE` fail-closed; list → `500`). Integration end-to-end remains BLOCKED until the Render deploy converges the schema.
- STALE/UNVERIFIED references intentionally kept out of binding sections: none. Any found later must be labelled.
