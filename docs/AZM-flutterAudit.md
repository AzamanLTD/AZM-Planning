**Azaman Flutter UI Audit — Shell, Navigation, Entry, HOME** 

**Read-only. All citations are file:line. Values are exact as written in source. Anything I could not verify is flagged [UNVERIFIED].** 

# **1. App shell** 

**Tab structure. Four tabs, hardcoded in main.dart:230-235 and main.dart:250-261:** 

- **idxlabel screen source 0 Home AzamanHomePage main.dart:231 1 Chat FriendsHubScreen main.dart:253** 

- **2 P2P P2PMarketplaceScreen main.dart:255** 

- **3 MarketMarketplaceHomeScreenmain.dart:257** 

**_pages is a List<Widget?> seeded with only index 0 non-null (main.dart:230-235); tabs 1-3 are lazily instantiated on first selection (main.dart:265) and then cached. Once built, all visited tabs stay alive in an Offstage stack (main.dart:441-446) — so state is preserved but every visited tab's widget tree, providers, and timers stay mounted forever. There is no AutomaticKeepAlive/dispose policy; a user who visits all four tabs holds four full screens in memory simultaneously.** 

**Bottom nav. PremiumBottomNav (premium_bottom_nav.dart:33) is a floating pill: height: 62 (:51), borderRadius: 31 (:54), color: colors.surface (:53), shadow Colors.black alpha 0.45 dark / 0.13 light, blurRadius: 24, offset (0,6) (:57-59). Outer padding fromLTRB(16, 0, 16, bottom>0 ? bottom+8 : 16) (:49). It is set as Scaffold.bottomNavigationBar with extendBody: true (main.dart:431-432), so content scrolls under it.** 

**Icons are HugeIcons stroke/solid pairs (:27-30): home01, message01, creditCard, store01. Selected color colors.accent, unselected colors.textTertiary (:112). Label fontSize: 10, weight w700 selected / w500 unselected, letterSpacing: -0.2 (:135-139).** 

**Badges (:165-196): tab 1 = numeric unread chat count; tab 2 = dot for active trades; tab 3 = dot for marketplace notifications. Badge color is hardcoded Color(0xFFEF4444) (:223, :237) — bypasses the theme's colors.danger. Numeric badge fontSize: 9, w800 (:228).** 

**Tab-switch animation. A single AnimationController _fadeCtrl with duration: MotionTokens.standard = 220ms (main.dart:237, motion_tokens.dart:29). The opacity curve is** 

**a hand-rolled V-shape (main.dart:281-285): fades to 0 by v=0.25, then back to 1 by v=1.0. The displayed index swaps at v>=0.25 via a listener (main.dart:238-242). So the outgoing tab fades out over ~55ms and the incoming fades in over ~165ms — an asymmetric crossfade, not a slide. Reduced-motion path sets _displayedIndex instantly and skips the controller (main.dart:266-278).** 

**Liquid indicator. LiquidTabBackdrop (liquid_tab_backdrop.dart:9) is a TweenAnimationBuilder keyed on selectedIndex (:46), 420ms, Curves.easeOutCubic (:4849). It computes a squash-and-stretch: stretch = 1 + 0.35*(1-|2t-1|), squash = 1 - 0.12*(1-| 2t-1|) (:53-54), rendering a 40px blob at color.withValues(alpha: 0.14) (:69), radius 20*squash (:70). Note the blob's own position uses Curves.easeOutCubic.transform(t) applied** **_again_ on top of the builder's alreadycurved t (:59) — a double-ease that makes the travel feel slightly frontloaded. [UNVERIFIED] whether this is intentional.** 

**Backdrop/gradient system. Two near-duplicate implementations:** 

 **ThemedAppBackdrop (themed_app_backdrop.dart:24) wraps the whole MaterialApp.router output (main.dart:205-207). Base LinearGradient topLeft→bottomRight from colors.background to alphaBlend(accent @ 0.06 dark / 0.030 light) (:36-46). Layer 1: radial glow top-left Alignment(-0.85,-0.95), radius 1.4, colors.glow @ 0.20/0.06 dark, 0.10/0.04 light, stops [0, 0.35, 1] (:55-64). Layer 2: radial accentSecondary bottom-right Alignment(0.95,1.0), radius 1.2, 0.12 dark / 0.06 light (:72-80). Layer 3 (light only): center wash colors.surface @ 0.80 (:85-100).** 

- **ThemedScaffold (themed_scaffold.dart:32) re-implements the** **_same_ three layers with slightly different alphas (glow 0.18/0.06 dark, 0.10/0.04 light at :8788; accentSecondary 0.10 dark at :104; center wash 0.95 at :121) and a base gradient at 0.04 dark / 0.025 light (:146). Two sources of truth for the same visual — a maintenance hazard and a subtle inconsistency (0.20 vs 0.18 glow, 0.12 vs 0.10 secondary, 0.80 vs 0.95 wash).** 

**AzamanHomePage itself uses a bare Scaffold(backgroundColor: Colors.transparent) (home_screen.dart:69-70), so it relies on the app-level backdrop.** 

# **2. HOME screen composition** 

# **AzamanHomePage (home_screen.dart:37) is a SingleChildScrollView with AlwaysScrollableScrollPhysics(parent: ClampingScrollPhysics()) and padding: EdgeInsets.only(bottom: 120) (:77-81). Top-to-bottom, in order:** 

|**#Section**|**Source**|**Visual treatment**|
|---|---|---|
|**1SizedBox(height: 8)**|**:85**|**—**|
|**2_GreetingHeader**|**:87,**<br>**def :122**|**Row: 46px gradient-ring avatar (Hero profile-avatar) +**<br>**Spacer + AZM rewards glass pill + NotificationBell + visibility**<br>**toggle.**|
|**3_GreetingTitle**|**:91,**<br>**def :303 **|<sup>**fontSize: 27, w800, letterSpacing: -0.5 (:318-323).**</sup>|
|**4_ActionPills**|**:95,**<br>**def :340**|**Horizontal scroll of 4 pills: Add Money / Send / Withdraw /**<br>**History.**|
|**5_BalanceCardsScroll**|**:99,**<br>**def :422 **|<sup>**SizedBox(height: 180) horizontal ListView of 3 cards.**</sup>|
|**6_SusuShortcutCard**|**:103,**<br>**def :571**|**Glass row w/ 56px progress**<br>**ring; renders SizedBox.shrink() unless an active susu**<br>**exists (:582).**|
|**7RecentActivitySection**|**ii:107**|**Section header + up to N transaction rows.**|
|**8LiveMarketSection**|**:111**|**"TODAY'S RATE" + USDC→GHS hero card + sparkline + rate-**<br>**alert row.**|



**Scroll depth. Rough vertical budget at 1x: 8 + ~46 (header) + 16 + ~34 (title) + 16 + ~70 (pills) + 18 + 180 (cards) + 28 + ~92 (susu) + 28 + ~(header 34 + 14 + 3×~62 rows ≈ 234) + 28 + ~(header 20 + 10 + hero ~200 + alert ~50 ≈ 280) + 120 bottom pad ≈ 1,150–1,300 logical px on a populated account. On a 6.1" phone (~800px viewport) that is ~1.5–1.7 screens of scroll. With an empty account (no susu, no txns) it collapses to ~700px. [UNVERIFIED] exact row count in RecentActivitySection — it renders txns.length rows unbounded (recent_activity_section.dart:87), so depth scales with the summary payload.** 

**Competing modules stacked. Eight distinct modules, of which five are "money/status" surfaces competing for the same glance: the balance card carousel (3 cards), the susu progress card, recent activity, and the live-rate hero. The balance carousel alone stacks three different card idioms side by side (home_screen.dart:436-454):** 

- **FlippableBalanceCard at screenWidth * 0.76 (:438)** 

- **_NewWalletCard at screenWidth * 0.38 (:446)** 

- **_MarketplaceShortcutCard at screenWidth * 0.38 (:451)** 

**Card geometry is inconsistent across the home screen. Radii in use: 22 (flippable/hologram back** 

**face flippable_balance_card.dart:270, hologram_balance_card.dart:32, _NewWalletCard hom e_screen.dart:475, _MarketplaceShortcutCard :530), 20 (susu card :597, dashboard card dashboard_balance_card.dart:131), 16 (live-market hero live_market_section.dart:165, onboarding use-case onboarding_screen.dart:471), 14 (today tiles today_widget.dart:258), 12 (settings tiles settings_drawer.dart:190). Heights: balance carousel fixed 180 (:431) vs HologramBalanceCard minHeight: 158 (hologram_balance_card.dart:29) vs DashboardBalanceCard aspectratio 1.586 (dashboard_balance_card.dart:49,81). Three different balance-card components exist (dashboard_balance_card.dart, hologram_balance_card.dart, flippable_balance_card.da rt) and the home screen uses only the flippable one.** 

**Dead code on home. TodayWidget (today_widget.dart:48) and QuickActionsRow (quick_actions_row.dart:11) are not referenced by home_screen.dart — verified by grep; TodayWidget is only mentioned in a comment in home_summary_service.dart:18. QuickActionsRow is referenced nowhere outside its own file. StoryRing is used in contacts/messages/friends/marketplace but not on home. So the home screen has three orphaned "home" widgets and no story rail, despite the app shipping a full story system.** 

# **3. Entry experience** 

**Splash (splash_screen.dart:24). Scaffold(backgroundColor: Colors.white) (:182) — hardcoded white, ignores theme entirely, so a dark-theme user gets a white flash. Content: LogoTraceLoader(size: 120, strokeWidth: 4, color: Colors.black @ 0.75, loopDurationSeconds: 2.4) (:193-198) + "AZAMAN" wordmark fontSize: 22, w700, letterSpacing: 8 (:216-221) with a ShaderMask vertical black gradient (:201-213) and a repeating shimmer 2200ms (:224-225). Entrance: AnimationController 800ms (:45), fade Curves.easeOut (:47), scale 0.85→1.0 Curves.easeOutCubic (:48-50).** 

**Then _checkAuthStatus (:69) does await Future.delayed(Duration(seconds: 2)) (:70) — a hard 2-second floor before any routing decision, regardless of how fast auth resolves. On web it additionally probes /health with a 3s timeout and silently flips to demo mode (:73-82).** 

**Routing is a chain of Navigator.pushReplacement(MaterialPageRoute(...)) (:101, 111, 145, 147, 151, 159, 163, 166, 169) — not GoRouter, so the splash→login→main path bypasses the router's transition system entirely and uses the default MaterialPageRoute (which is overridden by AzamanPageTransitionsBuilder via theme, so it does get the shared-axis slide — see §4).** 

**Onboarding (onboarding_screen.dart:27). 4 pages: 3 intro + 1 setup (:40). Intro pages use assets/images/1.webp, 2.webp, 3.webp (:45, 52, 60) in a ClipRRect(radius 24) Image.asset (:294-301). Top: LinearProgressIndicator minHeight: 3, radius 3, colors.accent (:133-140) + "Skip" TextButton (:145). Bottom: fullwidth ElevatedButton height: 52, radius 14, colors.accent (:204-219). Page transition: PageController.nextPage(duration: 400ms, curve: Curves.easeInOutCubic) (:105108). Dots: AnimatedContainer 300ms Curves.easeInOut, active width 24 vs 8, height 8, radius 4, with a glow shadow blurRadius 8 (:555-573). Setup page: 4 use-case cards, AnimatedContainer 200ms (:465), selected border 1.5 vs 1 (:476), AnimatedSwitcher 200ms for the check icon (:518-523). On completion it pushAndRemoveUntil to MainNavigationWrapper then, after 700ms, forcepushes DepositScreen (:92-100) — an aggressive post-onboarding deposit prompt.** 

**Login (login_screen.dart:20). Two-step flow (_step 0/1, :42). Step 0: Hero azaman_logo (:267) with a TweenAnimationBuilder shrinking 100→80 over 600ms Curves.easeOutCubic (:269272); "Welcome to Azaman" fontSize: 28 bold (:288-292); email + password TextFields with fillColor: colors.card, radius 12, BorderSide.none, focused border accent @ 0.5 (:322336); Login button height: 52, radius 12 (:369-380); "or continue with" divider; Apple (black) + Google (white) SSO buttons height: 50, radius 12 (:757-772); Sign Up link; "Try Demo (No Account Needed)" OutlinedButton (:474-496); influencer-code link (:513-524). Step 1 is an influencer-code screen (:529). Step transition: _fadeController 500ms Curves.easeInOut (:4751), reverse→swap→forward (:68-80). SSO is not actually wired — it shows an AlertDialog explaining Firebase isn't compiled in (:703-745).** 

**Signup (signup_screen.dart:16). Standard AppBar "Create Account" (:185193), Icons.swap_horiz at size: 64 as the hero icon (:201) — the same generic swap icon used for the P2P tab, a weak brand moment. Password strength bar with hardcoded colors 0xFFEF4444 / 0xFFF59E0B / 0xFF10B981 / 0xFFE5E7EB (:606-618). Same SSO-notconfigured dialog (:496-538).** 

# **4. Motion inventory** 

**Tokens (motion_tokens.dart): instantFeedback 90ms, microInteraction/fast 120ms, control 180ms, standard 220ms, emphasized 350ms, spatial 450ms, ambient 900ms, celebration 1200ms (:17-41). Curves: enter easeOutCubic, exit easeInCubic, spring easeOutBack, symmetric easeInOutCubic, decelerate (:45-61). Stagger 40ms step, 300ms cap (:65-68).** 

**Router transitions (transitions.dart): horizontalPage 300ms fwd / 250ms rev (:2728); fadeThroughPage 250/200 (:51-52); verticalPage 350/300 (:75-76); legacy sharedAxisPage 300/300 (:99-100). All bail to child when MediaQuery.disableAnimations (:30, 54, 78, 102).** 

**Global page builder (azaman_page_transitions.dart:22): inbound slide Offset(0.06,0)→0 + fade, outbound 0→Offset(-0.04,0) + dim 1.0→0.92, curves MotionTokens.enter/exit (:38-61). Reduced-motion → crossfade only (:34-36).** 

**Nav helpers (nav_transitions.dart): _SharedAxisRoute uses MotionTokens.standard (220ms) fwd / fast (120ms) rev (:78-79), Curves.easeOutCubic (:85). Vertical slide Offset(0,0.06) (:98), horizontal Offset(0.06,0) (:112), scaled 0.92→1.0 (:125).** 

**Home screen (home_screen.dart), all flutter_animate:** 

- **Section entrance stagger: header fadeIn 320ms (:87); title fadeIn 360ms + slideY(0.1) delay 100ms (:91); pills fadeIn 350ms + slideY(0.15) delay 200ms (:95); balance cards fadeIn 400ms + slideY(0.15) delay 300ms (:99); susu fadeIn 400ms + slideY(0.1) delay 400ms (:103); recent activity delay 500ms (:107); live market delay 600ms (:111). Total entrance cascade ≈ 1,000ms.** 

- **Avatar: fadeIn 300ms + scale(0.8→1.0) 300ms Curves.easeOutBack (:201-206).** 

- **AZM gift icon: infinite repeat(reverse: true) scale 1.0→1.1 800ms easeInOut (:227-233).** 

- **Visibility toggle: AnimatedSwitcher 250ms with RotationTransition (0.5→1.0 turns) + ScaleTransition (:267-274).** 

- **Action pills: per-pill fadeIn 300ms + slideY(0.2) staggered i*80ms (:367-375); each pill icon has an infinite shimmer 2000ms (:402-406).** 

- **_NewWalletCard: infinite shimmer 3000ms (:502-506).** 

- **_MarketplaceShortcutCard: fadeIn 400ms + slideX(0.2) delay 200ms (:565-567).** 

- **Susu ring: scale(0.5→1.0) 400ms easeOutBack (:627-628); card fadeIn 400ms + slideY(0.1) delay 300ms (:654-656).** 

**Balance cards: FlippableBalanceCard flip AnimationController 520ms Curves.easeInOutCubic (flippable_balance_card.dart:59-63), rotateX with perspective setEntry(3,2,0.001) (:164-166). HologramBalanceCard entrance fadeIn 380ms + slideY(0.04) (hologram_balance_card.dart:174-178); AnimatedNumber 600ms (:117) . DashboardBalanceCard titanium sweep 6s infinite (dashboard_balance_card.dart:115-118); live pulse dot 2s infinite reverse (:640-643); oracle countdown Timer.periodic(1s) (:530).** 

**Nav: tab crossfade 220ms (main.dart:237); liquid blob 420ms (liquid_tab_backdrop.dart:48); icon TweenAnimationBuilder MotionTokens.fast (120ms) easeOutBack (premium_bottom_nav .dart:123-126); label AnimatedDefaultTextStyle 120ms easeOut (:132-134); icon swap AnimatedSwitcher 120ms (:151-155).** 

**Sections: RecentActivitySection header fadeIn 300ms delay 100ms (:78); rows fadeIn 280ms + slideY(0.08) staggered i*80ms (:89-91). LiveMarketSection has no entrance animation — it appears instantly, inconsistent with every sibling. TodayWidget count AnimatedSwitcher 280ms with slide Offset(0,0.15) (today_widget.dart:288-299); header spinner AnimatedSwitcher 250ms (:210-211). NotificationBell fadeIn 300ms (notification_bell.dart:92) + badge TweenAnimationBuilder 800ms easeInOut infinite reverse (:48-51, 87).** 

**Entry: splash 800ms (splash_screen.dart:45), logo trace loop 2.4s (:197), shimmer 2200ms (:225); onboarding page 400ms (onboarding_screen.dart:106), dots 300ms (:556), use-case 200ms (:465), check 200ms (:519); login step 500ms (login_screen.dart:49), logo morph 600ms (:271).** 

**Reduced-motion coverage is partial. Respected in: router transitions (transitions.dart:30,54,78,102), global page builder (azaman_page_transitions.dart:34), nav helpers (nav_transitions.dart:82), tab crossfade (main.dart:266), nav button scale/label/icon (premium_bottom_nav.dart:113,125,133,150). Not respected in: every flutter_animate call on the home screen, the infinite shimmers, the 6s titanium sweep, the 2s pulse dot, the notification-bell pulse, the splash/onboarding/login animations. MotionTokens.accessibleDuration exists (motion_tokens.dart:79) but is not used by any of these.** 

# **5. Concrete weaknesses** 

**Overloaded home. Eight modules, five of them money/status surfaces, ~1.5–1.7 screens of scroll, and a 1,000ms entrance cascade before the last section is even visible. The balance carousel puts three unrelated card idioms in one horizontal strip (home_screen.dart:436-454).** 

**There is no single dominant "what is my money doing" answer — the user must parse a 3- card carousel, a susu ring, a transaction list, and a rate hero.** 

**Inconsistent card geometry. Radii 12/14/16/20/22 all in use across home + adjacent surfaces (see §2). Three separate balance-card components (dashboard_balance_card.dart, hologram_balance_card.dart, flippable_balance_card.dart) with three different sizing strategies (aspect ratio 1.586, minHeight 158, fixed parent 180). Two duplicate backdrop implementations with divergent alphas (themed_app_backdrop.dart vs themed_scaffold.dart).** 

**Hardcoded colors bypassing the theme. Color(0xFFEF4444) for nav badges (premium_bottom_nav.dart:223,237); Color(0xFF02C076) for success snackbars (login_screen.dart:163, signup_screen.dart:148); Color(0xFFFFD700) story ring (story_ring.dart:46); Color(0xFF161618)/0xFF0B0B0D/0xFF000000 card gradient (dashboard_balance_card.dart:158-160); password-strength palette (signup_screen.dart:606618); Colors.white splash scaffold (splash_screen.dart:182); Colors.black "Add" button text (recent_activity_section.dart:285). None of these respond to theme switching, and several (0xFFEF4444, 0xFF02C076) duplicate semantic roles the theme already defines as danger/success.** 

**Dead code. TodayWidget (today_widget.dart:48) and QuickActionsRow (quick_actions_row.dart:11) are unreferenced by the home screen (grep-verified). _BalanceNumber in hologram_balance_card.dart:182-284 is a fullyimplemented stateful animated-number widget that is never instantiated — the card uses AnimatedNumber instead (:112). _RoutedSurface in today_widget.dart:37-46 is a trivial pass-through wrapper around RoutedTabSurface. MainNavigationWrapper (main.dart:458) is a one-line alias for MainWrapper. _publicRoutes in auth_guard.dart:19-22 is declared but never read.** 

**Competing design idioms. Three navigation systems coexist: GoRouter (app_router.dart), raw Navigator.push(MaterialPageRoute(...)) (used throughout home, splash, login, signup, settings drawer), and the custom pushWithVerticalTransition helpers (nav_transitions.dart). The home screen mixes all three: Navigator.push for profile/rewards (home_screen.dart:144, 211), pushWithVerticalTransition for pills (:348-354), and context.push for susu (:592). Splash and auth use MaterialPageRoute exclusively, bypassing GoRouter. app_router.dart:122 explicitly documents this as intentional, but the result is that deep-link and in-app navigation have different transition and back-stack semantics.** 

**Reads as stock Material. RoutedTabSurface (routed_tab_surface.dart:29-47) is a default AppBar + Icons.arrow_back at size: 18. SignUpScreen uses a stock AppBar (signup_screen.dart:185) and Icons.swap_horiz at 64px as its hero (:201).** 

**Login/signup TextFields are default OutlineInputBorder with BorderSide.none (login_screen.dart:328-331). The SSO-notconfigured AlertDialog (login_screen.dart:709) is unstyled Material. _DisputeScreen (app_router.dart:728-757) is a bare AppBar + centered icon.** 

**Performance risks.** 

- **Unbounded rebuild** 

**surface. AzamanHomePage.build watches themeProvider (home_screen.dart:67) — the whole page rebuilds on any theme notify. _GreetingHeader watches authProvider (:128) and balanceVisibleProvider (:129); _GreetingTitle watches authProvider (:309). Hologr amBalanceCard watches balanceDataProvider + authProvider (hologram_balance_card .dart:19-20) and nests a Consumer for rate/currency (:93). LiveMarketSection watches homeSummaryProvider + rateHistoryProvider (live_ market_section.dart:69-71). RecentActivitySection watches homeSummaryProvider (r ecent_activity_section.dart:37). Because homeSummaryProvider is a single StateNotifier holding the entire snapshot (home_summary_provider.dart:24), any section's data change rebuilds every consumer of it — there is no select() partitioning on the home screen (contrast with dashboard_balance_card.dart:76, 298, 406 which does use select() correctly).** 

- **Infinite animations always running. At least five repeat() animations are live whenever home is mounted: AZM gift pulse (home_screen.dart:227), 4× pill shimmer (:402), wallet-card shimmer (:502), notification-bell pulse (notification_bell.dart:87), plus the 6s titanium sweep and 2s pulse dot if DashboardBalanceCard is mounted. None pause when the tab is Offstage (main.dart:443) — Offstage stops painting but does not stop tickers, so a user on the Chat tab still burns CPU on home's shimmers.** 

- **Large single-file widgets. settings_drawer.dart is 1,264 lines; login_screen.dart 794; home_screen.dart 661; signup_screen.dart 627; onboardi ng_screen.dart 576; live_market_section.dart 509; today_widget.dart 481. _GreetingH eader alone is ~180 lines of inline layout (home_screen.dart:122-301).** 

- **Timer.periodic in _OracleRefreshPill ticks every second and calls setState (dashboard_balance_card.dart:530-536) — a 1Hz rebuild of that subtree for a countdown that only needs to update once per second at most, and it runs even when the card is off-screen.** 

- **oracleRateProvider starts a Timer.periodic(60s) and a Future.microtask(refresh) at provider construction (hologram_provider.dart:201-203) — a network poll that runs for the app's lifetime once anything reads it.** 

# **Accessibility gaps.** 

- **Zero Semantics, semanticLabel, Tooltip, or ExcludeSemantics in any of the audited shell/home widgets (grep-verified** 

**across home_screen.dart, premium_bottom_nav.dart, notification_bell.dart, recent_ac tivity_section.dart, live_market_section.dart, flippable_balance_card.dart, hologram_ balance_card.dart). Icon-only controls — the visibility toggle (home_screen.dart:253285), the notification bell (notification_bell.dart:26), the nav icons (premium_bottom_nav.dart:116) — have no accessible labels.** 

- **Touch targets below 48dp. Nav icons are size: 22 inside a 62px bar split 4 ways (~90px wide, so width is fine, but the tappable Column is icon+4+label ≈ 40px tall). The visibility toggle is 40×40 (home_screen.dart:264-265). The notification bell is 40×40 (notification_bell.dart:33). The balance-card eye icon is size: 13 with EdgeInsets.symmetric(horizontal: 4, vertical:** 

   - **2) (dashboard_balance_card.dart:432-439) — roughly 21×17px. The _BalanceLine icon is 22×22 (flippable_balance_card.dart:368-369).** 

- **Contrast risks. colors.textTertiary is used for 9–11px labels throughout (home_screen.dart:412, live_market_section.dart:86, today_widget.dart:318). Colors. white.withValues(alpha: 0.45) for the UID line at fontSize:** 

**9.5 (dashboard_balance_card.dart:328) and alpha: 0.50 for the "TOTAL PORTFOLIO VALUE" label at 9.5 (:418) are almost certainly below 4.5:1 on the near-black card. [UNVERIFIED] without a contrast measurement, but the alpha values are low enough to flag.** 

- **Reduced-motion is only partially honored (see §4) — the infinite shimmers and pulses ignore disableAnimations entirely.** 

- **_ActionPills is a horizontal SingleChildScrollView (home_screen.dart:356) with no scroll affordance or semantics hint that more content exists off-screen.** 

# **6. Verdict vs. Phantom / Revolut** 

**Structure. Phantom and Revolut both lead with a single, unambiguous primary surface: one balance, one primary action set, and a short, scannable feed. Azaman's home instead presents eight modules and three balance-card components, with the balance itself split** 

**across a flippable card, a wallet card, a marketplace card, a susu ring, and a rate hero. The information architecture is additive rather than curated — it reads like every sprint appended a section without removing one. The presence of two orphaned home widgets (TodayWidget, QuickActionsRow) and an unused animated-number widget confirms that sections were built and then abandoned rather than consolidated.** 

**Restraint. This is the biggest gap. Phantom's motion is sparse and purposeful; Azaman runs at least five infinite animations on the home screen alone (gift pulse, 4× pill shimmer, wallet shimmer, bell pulse, plus the card sweep and pulse dot), and its entrance cascade takes a full second. The flutter_animate shimmer on every action pill (home_screen.dart:402-406) is decorative noise with no informational value. The 6-second titanium sweep and 1Hz oracle countdown (dashboard_balance_card.dart:115, 530) are ambient effects that a top-tier app would cut or gate behind reduced-motion.** 

**Craft. The motion token system (motion_tokens.dart) is genuinely good and better than most production Flutter apps — but it is only partially adopted: the home screen hardcodes durations (320ms, 360ms, 350ms, 400ms, 500ms, 600ms) instead of pulling from tokens, and reduced-motion is honored in the router but ignored in the widgets. The theme system (AzamanColors) is well-structured, but hardcoded hex values leak through in nav badges, snackbars, the story ring, the balance card gradient, and the splash screen. The backdrop system is duplicated across two files with divergent values.** 

**Where it's competitive. The LiquidTabBackdrop squash-and-stretch (liquid_tab_backdrop.dart:53-54) is a genuinely nice, cheap touch that reads as more considered than a stock NavigationBar. The FlippableBalanceCard flip-to-breakdown (flippable_balance_card.dart:144-181) is a real product idea — surfacing escrow/vault/susu/savings locked balances on the back of the card is the kind of thing Revolut does well. The select()-based granular rebuilds in dashboard_balance_card.dart show the team knows how to do this correctly.** 

**Honest summary. The** **_ingredients_ are close to top-tier — good motion tokens, a real theme system, thoughtful individual widgets. The** **_composition_ is not. Azaman's home screen is roughly 1.5–1.7 screens of stacked, competing modules with inconsistent card geometry, five always-on animations, partial reduced-motion support, and no accessibility semantics on any icon-only control. Against Phantom or Revolut, the gap is not visual polish — it is editorial discipline: those apps ship fewer surfaces, each with one clear job, and they cut motion that doesn't carry information. Azaman has the opposite problem: it ships every surface and animates all of them.** 

**Highest-leverage fixes, in order: (1) collapse the three balance-card components into one and remove the two orphaned home widgets; (2) gate all infinite animations** 

**behind MediaQuery.disableAnimations and delete the decorative pill shimmer; (3) add Semantics labels to the icon-only controls and raise sub-48dp targets; (4) replace hardcoded hex with theme tokens; (5) partition homeSummaryProvider with select() so a rate tick doesn't rebuild the transaction list.** 

# **AZAMAN MARKETPLACE — UI/UX TECHNICAL AUDIT (read-only)** 

Root: C:\Users\User\.verdent\verdent-projects\Azaman\main\AZM-frontend-main Scope: Marketplace system only. All claims verified in code with file:line. Items I could not verify are flagged **[UNVERIFIED]** . 

# **0. SHARED FOUNDATION (applies to all four verticals)** 

# **0.1 Theme system — lib/providers/theme_provider.dart** 

- AzamanColors (lines 271–326) is the canonical palette. Only **two** themes exist (lines 19– 27): light and dark. 

   - Light: background #FAFAFB, surface #FFFFFF, card #FFFFFF, softSurface #F1F1F3, divider #E6E6E9, accent #B8860B (dark gold), accentSecondary #8B6914, accentSurface #FDF6E3, success #018C5C, danger #D32C44, warning #C78A00, textPrimary #111827, textSecondary #374151, textTertiary #6B7280, glow #B8860B, border #E6E6E9 (lines 208–229). 

   - Dark: background #000000 (true black), surface #0A0A0A, card #161616, softSurface #0F0F0F, divider 0x12FFFFFF (white 7%), accent #2DD4BF (teal), accentSecondary #F59E0B (amber), accentSurface 0x1A2DD4BF, success #34D399, danger #F87171, warning #FBBF24, textPrimary white, textSecondary white70, textTertiary white38, glow #2DD4BF, border 0x14FFFFFF (lines 239–260). 

- ThemeData (lines 111–203): scaffoldBackgroundColor: Colors.transparent (line 120) so a ThemedAppBackdrop gradient shows through; fontFamily: 'Inter' (line 122); global PageTransitionsTheme = AzamanPageTransitionsBuilder on all platforms (lines 126 –135); elevatedButtonTheme radius **12** (line 175); inputDecorationTheme radius **12** (lines 182–189); snackBarTheme floating (lines 155–159). 

- **Weakness** : colorScheme (lines 191–201) omits tertiary, surfaceContainer*, outline, shadow, scrim, inverseSurface — any Material 

3 widget that reads those falls back to Flutter defaults, which will not match the Azaman palette. 

# **0.2 Motion system — lib/theme/motion_tokens.dart** 

- Durations: instantFeedback 90ms, microInteraction 120ms, control 180ms, standard 220ms, emphasized 350ms, spatial 450ms, ambient 900ms, celebration 1200ms. 

- Curves: enter = easeOutCubic, exit = easeInCubic, spring = easeOutBack, symmetric = easeInOutCubic. 

- accessibleDuration(context, d) returns Duration.zero when MediaQuery.disableAnimations is true. 

 **Weakness** : adoption is partial. service_experience_stage.dart uses MotionTokens.accessibleDuration (lines 98, 140, 226) and marketplace_vertical_experience_stage.dart uses MotionTokens.enter/ exit (line 85), but most marketplace screens hardcode raw Duration(milliseconds: …) + Curves.* instead (see per-vertical sections). The token system is not enforced. 

# **0.3 Glass primitive — lib/widgets/premium_glass_container.dart** 

- Defaults: blur 20, opacity 0.08, borderRadius 20. Marketplace call sites override to blur 12, opacity 0.04/0.06/0.08, radius 14/16/18 — i.e. **no two call sites agree** , and none reference a shared constant. 

# **0.4 The "experience" abstraction (the answer to question 4)** 

There **is** a real shared abstraction, but it is a _routing/control plane_ , not a shared visual language. 

 MarketplaceExperienceBlueprint.fromJson(experience, category) — lib/marketplace/experiences/marketplace_experience_blueprint.dart:55– 100 — parses server-driven UI JSON into a blueprint with: preset, navigationMode, detailPresentation, commitStyle, persistentTray, motionTe mpo, customerContext, etc. 

- Presets: DINING_JOURNEY, SHOP_FLOOR, BUILDING_WALK, TRAVEL_JOURNEY, plus SERVICE_JOURNEY default. 

- _MarketplaceCategoryPolicy (lines 127–173) whitelists which navigation modes / detail presentations / commit styles / customer-context flags each category may use — this is the single most important shared contract. 

- MarketplaceVerticalExperienceStage.build() — lib/widgets/marketplace/ marketplace_vertical_experience_stage.dart:52–86 — switches on blueprint.preset and 

routes to a per-vertical stage; wraps the result in AnimatedSwitcher(duration: blueprint.motionDuration(context), switchInCurve: MotionTokens.enter, switchOutCurve: MotionTokens.exit, key: ValueKey('${preset}:${motionTempo}')) (line 85). 

- _legacyStage() (lines 88–104) is a **dead fallback** to the older MarketplaceExperienceCatalog/MarketplaceExperienceCapability path — superseded by presets. 

 MarketplaceDetailSurface — lib/widgets/marketplace/ marketplace_detail_surface.dart:11–176 — is the one genuinely shared _visual_ component: it implements 6 MarketplaceDetailPresentation variants behind one widget. Its only hardcoded value is a drag-handle pill colors.textTertiary.withValues(alpha:0.35) radius 999 (line 176) — correctly themedriven. 

**Verdict on sharing** : the four verticals share (a) the blueprint/policy control plane, (b) MarketplaceDetailSurface, (c) MarketplaceVerticalExperienceStage routing, (d) AzamanColors/MotionTokens as _available_ tokens. They do **not** share a visual language: each vertical has its own bespoke stage, its own card geometry, its own radii, and its own hardcoded palette. Retail is Theme.of(context)-driven; Restaurant is a hardcoded warm-paper metaphor; Hotel and Transit are bespoke screens that bypass the stage entirely in places. 

# **1. RETAIL (SHOP_FLOOR)** 

# **1.1 Screen flow** 

marketplace_home_screen.dart feed → CollapsibleBusinessBar (collapsed → expanded) → context.push('/business/{bizId}') (collapsible_business_bar.dart:117–120) 

→ business_profile_screen.dart → BusinessBookTab (business_book_tab.dart:18) 

→ MarketplaceVerticalExperienceStage → retail stage. Also reachable via catalog_storefront_screen.dart and the SDUI storefront/ renderer. 

# **1.2 Layout structure & concrete values** 

- **CollapsibleBusinessBar collapsed bar** (collapsible_business_bar.dart:221–402): height **90** (line 232), margin symmetric(vertical:5, horizontal:2) (line 231), radius **16** (line 245), gradient cat.color @ alpha 0.08 dark / 0.05 light → colors.card, stops [0.0, 0.35] (lines 236–244). Two box shadows: cat.color @ 0.12, blur 16, offset (0,4) and black @ 0.22 dark / 0.07 light, blur 14, offset (0,4) (lines 246–257). Left image tile **90×90** (lines 266–277). Category label overlay: cat.color @ 0.82, white text, fontSize 

8, w900, letterSpacing 0.4 (lines 283–297). Name fontSize 16 w800 (line 316), rating stars size 11 with filledColor Color(0xFFF59E0B) (line 336), subtitle fontSize 11.5 textTertiary (line 374). Chevron AnimatedRotation(turns: isExpanded ? 0.25 : 0, 300ms, easeOutCubic) (lines 389–392). 

- **AnimatedSize** wraps collapsed↔expanded: 280ms, Curves.easeInOut (lines 202–204). 

- **_ExpandedAvatar** (lines 1074–1175): 52×52 squircle, radius 16, ContinuousRectangleBorder clip, border colors.card width 2.5, shadow black @ 0.15, blur 8, offset (0,3). 

- **_ExpandableProfilePic** (lines ~1001–1066): flush rectangle, Hero(tag: 'biz-pic-${bizId}'), lightbox via PageRouteBuilder(opaque:false, barrierColor: Colors.black87) + FadeTransition (lines 1008–1034). 

- **Retail experience** 

**widgets** (lib/marketplace/experiences/retail/retail_experience.dart): RetailCollectionBox , RetailProductCard, RetailQuickLookSheet, RetailCartSheet — all driven by Theme.of(context), **not** AzamanColors. 

# **1.3 Animations** 

AnimatedSize 280ms easeInOut; AnimatedRotation 300ms easeOutCubic; Hero lightbox fade; flutter_animate used 

in marketplace_status_rail.dart (.animate().scale().fadeIn().shimmer()). catalog_storefront_scree n.dart: 380ms easeOutCubic (lines 81–82), 160ms easeOut (line 90), 220ms easeOut (lines 136– 137), radius 20 (line 142). 

# **1.4 Weaknesses** 

- **Hardcoded colors bypassing theme** : category_filter_panel.dart:16–29 defines 8 hardcoded hex category colors; lines 183–195 

use Colors.grey.shade300/600/700. business_card.dart:123 Color(0xFF1e8449) OpenNow chip, :126 Color(0xFF1a6b8a) Verified 

chip. collapsible_business_bar.dart:336 and :709 Color(0xFFF59E0B) star fill. marketplace_home_screen.dart:1730 Color(0xFFF59E0B). 

- **Two cart systems coexist** : app-wide cartProvider/FloatingCartBar/CartScreen vs the standalone RetailCart/RetailCartSheet inside the storefront widget. No shared model. 

- **Two theming systems** : AzamanColors vs StorefrontThemeResolver (see §5). 

- **Stock-Material look** : catalog_storefront_screen.dart uses a plain SliverPersistentHeaderDelegate (_StorefrontCategoryBarDelegate, lines 210–223) with no visual treatment. 

- **Empty/loading/error** : FeaturedProductsSection silently returns SizedBox.shrink() on empty (featured_products_section.dart:103) and its _shimmerRow (lines 232–250) is a non-animated static row — not a real skeleton. 

# **2. RESTAURANT / DINE-IN (DINING_JOURNEY)** 

# **2.1 Screen flow** 

dinein_tab_screen.dart (list of restaurants, 10s auto-refresh Timer.periodic, line 34) → dinein_restaurant_ordering_screen.dart → BusinessBookTab (business_book_tab.dart:18) → MarketplaceVerticalExperienceStage → RestaurantNativeMenuJourney → FlipBook. Also business_profile_screen.dart "Book" tab reuses the same BusinessBookTab. 

# **2.2 Layout structure & concrete values** 

- **RestaurantMenuFlipBook** (lib/widgets/restaurant_menu_flip_book.dart): hardcoded palette at lines 29–34 — _paperEven #FDF6E3, _paperOdd #F5EDE0, _ink #2D2416, _inkSoft #8B7A5A, _gold #B8860B, _rule #E8DCC4. _itemsPerPage = 4 (line 110). Book padding fromLTRB(16,14,16,8) (line 194). BookMaterial(paper: _paperEven, castShadow: #1A1206, edge: #D8C6A2) (lines 202–206). 

- **_AmbientStage** (lines 49–81): RadialGradient #1F1409 alpha 0→0.82, radius 0.85, over ImageFiltered(blur sigmaX/Y 14, TileMode.mirror) of assets/images/food/restaurant_interior.jpg. 

- **_pageSurface** (lines 300–318): even pages gradient [#F3E9D2, _paperEven, #FFFBF0] stops [0.0,0.42,1.0]; odd [#EDE3CD, _paperOdd, #FDF7E8]; padding fromLTRB(26,24,20,16). 

- **Cover page** (lines 320–387): hero image 120×120 radius 20, name fontSize 20 w800 letterSpacing -0.3, "MENU" fontSize 13 w800 letterSpacing 4 in colors.accent, dish count fontSize 11. 

- **Dish row** (lines 430–506): 46×46 image radius 10, name fontSize 12.5 w700, price fontSize 11.5 w800 in _gold, veg dot Colors.green 8×8 (line 464), spicy icon Color(0xFFE05A33) (line 497), "Popular" chip colors.accent @ 0.1 radius 4 (lines 475 –482). 

- **Pop-out overlay** (lines 508–629): TweenAnimationBuilder 220ms easeOutCubic, scrim black @ 0.55*t, Transform.scale(0.92 + 0.08*t), card radius 20, shadow black @ 0.35 blur 30 offset (0,12), image height 170, "Add to order" ElevatedButton radius 12. 

- **RestaurantCommitSurface** (lib/widgets/marketplace/restaurant_commit_surface.dart): _PaperRipPainter (lines 225–354) with hardcoded #D9C7A5 seam (line 270), #2D2416 text (line 301), #8B7A5A (line 312), #E5D4B5 fold (line 342). Commit durations **820ms relaxed / 720ms balanced / 560ms quick** (lines 44– 

53), _commitDelay = 39% of duration; respects MediaQuery.disableAnimations (line 83) with a reduced-motion toast fallback. 

 **RestaurantNativeMenuJourney** (lib/widgets/marketplace/ restaurant_native_menu_journey.dart): Colors.black @ 0.34 (line 127), paper #FDF6E3/#D8C6A2 (lines 145–149), page text #2D2416/#8B7A5A/#B8860B/#B8A88A (lines 200–253). 

# **2.3 Animations — the flip-book engine (lib/widgets/book/)** 

This is the most sophisticated animation work in the app. 

- **FlipPhysics** (flip_physics.dart): FlipPhysicsSpec defaults stiffness 190, dampingRatio 0.86, mass 1.0, velocityTransfer 1.0, decelerationFactor 0.135, deadZone 0.012 (lines 45– 52). hint spec: stiffness 120, dampingRatio 1.0, velocityTransfer 0.0 (lines 58–62). Release resolved by _projected landing_ (progress + velocity * decelerationFactor >= 0.5), not a naive 0.5 test (lines 90–102). rubberBand overscroll curve (lines 136–139). 

- **FlipBookController** (flip_book_controller.dart): state machine idle/dragging/settling/hinting (line 18). edgeAnchored restricts grabs to the outer 58% width (lines 101–107). Haptic selectionClick on threshold cross (line 296). playHint() = 520ms easeOutCubic out to 0.085 then 620ms easeInOutCubic back (lines 206–224). turnForward/Backward settle with velocity ±1.4 (lines 179, 192). 

- **PageCurlSolver** (page_geometry.dart): cylindrical curl, curvedSlabs 28, cameraDistanceFactor 2.6, light dir (-0.34,-0.55,0.76), ambient 0.72 (lines 78–83). Reuses Float32List/Int32List buffers across frames (lines 85–86, 377–381) — zero steadystate garbage. FlipPath.touchFor lift = sin(p·π)·h·0.055 (line 396). 

- **FlipBook** (flip_book.dart): RenderRepaintBoundary.toImageSync() GPU rasterization + drawVertices deformation. PageCurlPainter uses **no saveLayer** (page_curl_painter.dart: 16). BookMaterial defaults: paper #FBF3E0, castShadow #1A1206, edge #D8C6A2, bleed 0.18, specularStrength 0.35, shadowStrength 0.55, spineStrength 0.18 (lines 48–56). 

# **2.4 Weaknesses** 

- **Entirely hardcoded warm-paper palette** that ignores AzamanColors — in dark mode the menu stays cream/ink, which is a deliberate metaphor but breaks theme consistency. Files: restaurant_menu_flip_book.dart:29–34, restaurant_native_menu_journey.dart:12 7,145–149,200–253, restaurant_commit_surface.dart:270,301,312,342, page_curl_paint er.dart:48–56. 

- **Duplicated layout** : BusinessBookTab is reused by both the profile Book tab and dinein_restaurant_ordering_screen.dart — good — but dinein_tab_screen.dart (241 lines) re-implements its own restaurant list rather than reusing CollapsibleBusinessBar. 

- **dinein_tab_screen.dart** uses a 10s polling Timer.periodic (line 34) instead of a provider stream — battery/network cost, and no visible loading/error state in the file. 

- **dinein_restaurant_ordering_screen.dart** has a 1500ms hardcoded duration (line 100) not from MotionTokens. 

- **Stock-Material** : dinein_tab_screen.dart:223 RoundedRectangleBorder(radius 8) and :146 radius 999 — arbitrary values not in any scale. 

# **3. TRANSIT (TRAVEL_JOURNEY)** 

# **3.1 Screen flow** 

transit_trip_list_screen.dart → transit_seat_selection_screen.dart → BusSeatSelector (lib/ widgets/seat_selector/bus_seat_selector.dart) → transit_seat_preview.dart. 

# **3.2 Layout structure & concrete values** 

- **TransitTripListScreen** (transit_trip_list_screen.dart:20): _TripCard (line 81) radius **12** (line 168), inner chip radius **6** (line 175), a 1px divider radius 1 (line 145). **Has proper loading (SkeletonList), error (with Retry), and empty (icon + text) states** (lines 37–76) — the best state coverage in the marketplace. 

- **BusSeatSelector** (bus_seat_selector.dart): CustomPainter repaints via repaint: Listenable.merge([controller, pulseAnimation]) (line 415) — never setState during pan/zoom. SVG seat icons decoded once to a ui.Picture cache (lines 186–224). Auto-fit uses Curves.easeOutBack with a 350ms _fitController. Seat tap centers viewport at 1.5× zoom. Pulse ring = 1200ms repeating controller. Checkout dock animates total via TweenAnimationBuilder 400ms easeOutCubic with tabular figures (lines 626–644). Minimap only shows when _currentZoomScale > _autoFitScale * 1.15 (line 438). 

- **transit_seat_selection_screen.dart** (475 lines): has loading/error/empty states. 

# **3.3 Animations** 

easeOutBack 350ms auto-fit; 1200ms pulse; 400ms easeOutCubic total tween; AnimatedOpacity/AnimatedSwitcher in the dock. All hardcoded, none from MotionTokens. 

# **3.4 Weaknesses** 

- **Bespoke, not blueprint-driven** : the seat selector is a self-contained engine that does not consume MarketplaceExperienceBlueprint beyond preset routing. Its radii (12/6/1) and durations are local constants. 

- **transit_seat_preview.dart** duplicates a mini seat-map concept already in bus_seat_selector.dart. 

- **No shared empty/loading component** : SkeletonList here is local; other verticals roll their own or have none. 

# **4. HOTEL (BUILDING_WALK)** 

# **4.1 Screen flow** 

business_profile_screen.dart → BusinessBookTab → MarketplaceVerticalExperienceStage → hotel stage (HotelFloorPlanPreview) 

→ hotel_booking_screen.dart (_ShowcaseSlider → _RoomExplorer → _RoomTile → _RoomDeta ilCard → _BookingPanel). 

# **4.2 Layout structure & concrete values** 

- **hotel_booking_screen.dart** (594 lines): bespoke _ShowcaseSlider, _RoomExplorer, _RoomTile, _RoomDetailCard, _BookingPanel . **[UNVERIFIED]** exact radii/paddings inside these private widgets — I read the file but did not extract every numeric literal; the class inventory is confirmed. 

- **HotelFloorPlanPreview** (lib/widgets/marketplace/hotel_floor_plan_preview.dart): floor strips; hardcoded Color(0xFFF59E0B) deluxe accent at line 168. 

- **HotelRoomExplorer** (lib/marketplace/experiences/hotel/hotel_experience.dart) duplicates the room-tile concept from hotel_booking_screen.dart's _RoomExplorer. 

# **4.3 Animations** 

**[UNVERIFIED]** — I did not extract the specific durations/curves from hotel_booking_screen.dart's private widgets. The file is 594 lines and uses 

standard AnimatedContainer/TweenAnimationBuilder patterns, but I cannot cite exact values without re-reading. 

# **4.4 Weaknesses** 

- **Duplicated room-tile** 

**layout** across hotel_booking_screen.dart and hotel_experience.dart. 

- **Hardcoded #F59E0B** in hotel_floor_plan_preview.dart:168 (same amber as the star color used elsewhere — a de-facto un-tokenized brand color). 

- **Bespoke screen bypasses the shared stage** : hotel_booking_screen.dart is a standalone Scaffold-level screen, not composed from MarketplaceDetailSurface. 

# **5. STOREFRONT SDUI (cross-cutting, affects all verticals)** 

- **Second theming** 

**system** : StorefrontThemeResolver.resolve() (lib/storefront/core/storefront_theme_resol ver.dart) builds a full ThemeData from a 

remote ThemeTokenSet (storefront_models.dart:78–126). spacingScale clamped 0.5–2.0 (line 36); border radius mapped small=6 / medium=10 / large=16 (lines 141– 

148). ThemeTokenSet.accent defaults to #6C4FD1 (purple) 

at storefront_models.dart:118 — a **third** accent color alongside gold and teal. 

- **Widget registry /** 

**renderer** : lib/storefront/core/widget_registry.dart + storefront_renderer.dart map widge tType strings to widgets. StorefrontTrackingScope (storefront_tracking_scope.dart) is a clean InheritedWidget for 

analytics; StorefrontVisibilityDetector (storefront_visibility_detector.dart) fires widget_view on viewport entry using tile id (stable across reorder) — welldesigned. 

- **Fake/placeholder data in production** 

**widgets** : LiveStatsWidget hardcodes '1.2K', '348', '5.4K', '4.5' (lines 18– 21). ShowcaseGalleryWidget, SocialFeedWidget, LocationMapWidget, CustomHtmlWidg et, VideoPlayerWidget render placeholder 

content. ProductGridWidget has _EmptyProductState; ReviewCarouselWidget has _Empt yReviews; storefront_skeleton.dart exists for loading. 

- **Provider layer** (storefront_provider.dart): storefrontExperienceProvider (line 104) and storefrontProductsProvider (line 105) are the two families the marketplace verticals 

actually consume. DraftLayoutNotifier (lines 39–98) handles editor save/publish/conflict — not used by the consumer marketplace. 

# **6. CROSS-VERTICAL SUMMARY TABLE** 

|Concern|Retail|Restaurant|Transit|Hotel|
|---|---|---|---|---|
|Blueprint preset|SHOP_FLOOR|DINING_JOURNEY|TRAVEL_JOUR<br>NEY|BUILDING_WALK|
|Stage widget|retail stage|RestaurantNativeMen<br>uJourney|seat selector|HotelFloorPlanPre<br>view|
|Palette source|Theme.of(cont<br>ext)|hardcoded paper|local constants|<sup>AzamanColors + #</sup><br>F59E0B|
|Signature animation|AnimatedSize<br>280ms|flip-book spring<br>190/0.86|easeOutBack 3<br>50ms|<br>**[UNVERIFIED]**|
|Loading/error/empty|partial|none in tab|full|**[UNVERIFIED]**|
|Uses MotionTokens|no|no|no|no|
|Uses MarketplaceDeta<br>ilSurface|yes|yes|yes|no (bespoke<br>screen)|



# **7. TOP CONCRETE WEAKNESSES (ranked, all verified)** 

1. **Three competing accent colors** : gold #B8860B (light theme), teal #2DD4BF (dark theme), purple #6C4FD1 (storefront default) 

   - theme_provider.dart:217,248 + storefront_models.dart:118. 

2. **MotionTokens is not adopted** by any marketplace screen; every screen hardcodes its own durations/curves. 

3. **Restaurant vertical is fully hardcoded** and ignores dark mode — 6 files with warm-paper hex literals. 

4. **Two cart systems** (app-wide vs RetailCart) with no shared model. 

5. **Two theming systems** (AzamanColors vs StorefrontThemeResolver) with different radius scales (12 vs 6/10/16). 

# 6. **Duplicated** 

**layouts** : _RoomExplorer vs HotelRoomExplorer; transit_seat_preview vs bus_seat_select or; dinein_tab_screen list vs CollapsibleBusinessBar. 

7. **Dead code** : _legacyStage() (marketplace_vertical_experience_stage.dart:88– 104), _about() (business_profile_screen.dart:1267–1324), _hero()/_circleButton() (busin ess_profile_screen.dart:1076), _nearbyCard() (marketplace_home_screen.dart:1461). 

8. **Fake data shipped in production widgets** : LiveStatsWidget hardcoded metrics. 

9. **Silent empty states** : FeaturedProductsSection returns SizedBox.shrink() with no user feedback (featured_products_section.dart:103). 

10. **Incomplete ColorScheme** (theme_provider.dart:191–201) — M3 widgets fall back to Flutter defaults. 

# **8. EXPLICITLY UNVERIFIED** 

- Exact radii/paddings/durations inside hotel_booking_screen.dart's private widgets (_ShowcaseSlider, _RoomExplorer, _RoomTile, _RoomDetailCard, _BookingPanel) — class inventory confirmed, numeric literals not extracted. 

- Whether _about() / _hero() / _nearbyCard() are truly unreferenced — I inferred from reading, but did not run a full rg reference check across the whole repo. 

- The full contents of lib/storefront/widgets/* beyond the widgets named in §5 — I verified the named ones only. 

**No files were modified.** This was a read-only audit. 

# **AZAMAN Fintech UI Audit — Read-Only** 

Root: C:\Users\User\.verdent\verdent-projects\Azaman\main\AZM-frontend-main. No files modified. All values below are read from source; anything I could not verify is flagged **[UNVERIFIED]** . 

# **1. Core money-movement flows** 

# **Send money — lib/screens/send_money_screen.dart** 

Single screen, **zero confirmation steps** . Order: balance chip (accentSurface bg, radius 12) → "Send to" search field + 46×46 search button → recipient card (radius 14, accent@0.3 border) 

or _ContactsList (friendProvider) → amount field (radius 12, prefix USDC, fontSize 20) → optional note → inline banner (radius 10) → Send button (height 52, radius 14, fontSize 16, w700). Posts /finance/internal-transfer. Success = **inline green banner text only** — no sheet, no animation, no haptic celebration. This is the weakest success moment of the three flows. 

# **Deposit — lib/screens/deposit_screen.dart (1771 lines)** 

Single screen, 2 segmented tabs (_SegmentedTabs, UnderlineTabIndicator width **2.5** , label fontSize **16** w800, labelPadding L16/R16/B8, tab height 44, NoSplash.splashFactory). Crypto panel: Polygon USDC address card + copy/share + last-2 recent deposits. Mobile Money panel: saved MoMo tiles, "Add Account" pill (height **44** , 

radius **22** , accent@0.12 fill, accent@0.35 border 1.2), giant amount field fontSize **56** w800 letterSpacing **-1.0** , quick chips [50,100,200,500] (radius 22, 200ms AnimatedContainer), "Send deposit prompt" button. 

Step count is **variable and the worst offender** : form → optional Moolre namevalidation AlertDialog (_validateAndConfirm, lines 509–557) → OTP entry if requiresOtp (line 593) → result. So the user can face **1, 2, or 3** gates depending on backend response. Demo mode auto-confirms after Duration(seconds: 3) (line 609). Success = Lottie assets/animations/success.json at 110×110 + "Deposit Confirmed!" / "Waiting for confirmation..." with _PulsingDots. Socket-driven confirmation via SocketService.instance.onDepositSuccess (line 475) with a 5s floating snackbar. 

# **Withdrawal — lib/screens/withdrawal_screen.dart (1946 lines)** 

Single screen, 2 modes. Mode toggle is a **hand-rolled duplicate** of the deposit tab strip (_buildModeToggle, lines 842–894): AnimatedContainer 180ms, underline width **28** when selected / 0 when not, height **2.5** , radius 2 — visually near-identical to deposit's TabBar but implemented completely differently. Balance display fontSize **34** w800 letterSpacing **-0.8** (line 918). Amount field radius **16** , fontSize 24, softSurface fill, "Max" suffix. Fee preview: const baseRate = 0.02 (line 117) × (1 - discount). 

Uses **SlideToConfirm** + AzamanBiometricGate with GlobalKey<SlideToConfirmState> for reseton-cancel (lines 81–82). Success = snackbar + Navigator.pop (lines 274–278) — **no success screen at all** , the user is dumped back to the previous route. 

**Verdict on flow consistency:** three flows, three different confirmation philosophies (none / dialog+OTP / slide+biometric) and three different success treatments (inline text / Lottie screen / snackbar+pop). 

# **2. Wallet / vault / susu / rewards / trade surfaces** 

**Shared visual language is partial and split by background treatment.** Vault and susu screens use Scaffold(backgroundColor: Colors.transparent) so the ThemedAppBackdrop gradient shows through; deposit, send, withdrawal, transaction history, spending insights and credit-score use opaque colors.surface/colors.background. That single difference makes the two families read as different apps. 

- **Vault** (vault_list, vault_detail, vault_create, vault_yield, shared_vault): the most designed family. VaultProgressCard uses BackdropFilter blur 14 + colors.card @ alpha 0.55 + hardcoded const rate = 0.08 projected 

yield. vault_create _RulesSheet uses BackdropFilter blur 18, height 78% of screen, scrollto-bottom + consent checkbox gate. vault_yield enabled gradient is hardcoded 0xFF1B5E20/0xFF2E7D32 — a green that exists in no theme token. 

- **Susu** (hub, dashboard, config, completion, credit_score, position_picker): heaviest stockMaterial 

leakage. susu_completion _MemberPayoutRow and susu_position_picker _MemberRow are raw ListTiles; susu_config uses ChoiceChip + showDatePicker. susu_credit_score tier colors are hardcoded 0xFF00D97E/0xFF00B4D8/0xFFFF9500/0xFFFF3B30 and the sparkline is _generateHistorySpots **fake noise** — the score itself is simulated, not backend data. susu_position_picker contains dead Taylor-series trig helpers (_dartCos/_dartSin/_mathCos/_mathSin) that are never meaningfully used. 

- **Rewards** (azm_rewards_screen, referral_screen, round_up_settings_screen): azm_rewar ds uses ExpansionTile for earn-rates (stock Material). _AzmProgressBar (p2p) hardcodes 0xFFD4AF37 and animates 0→target over **900ms easeOutCubic** with milestones Trader I@50, II@100, Verified@250, Power@500, Elite@1000, Diamond@2500, Legend@5000. 

 **Trades** (trades_tab_screen, active_trade_screen 1811 lines, trade_summary_screen, trade_accounts_screen): trades_tab_screen uses a stock TabBar/TabController. active_trade_screen is the most feature-dense surface (chat + timer + dispute + escrow) and the most visually inconsistent: _buildLaunchButton (line 1415) and _buildBottomAction (line 1582) are raw ElevatedButtons with minimumSize: Size(double.infinity, 55) and radius **12** , while _buildTimerHeader (line 1437) uses fontFamily: 

'monospace' and AnimatedContainer 300ms. _buildDisputedInterface (line 1508) uses a 56px icon in a danger@0.1 circle. _RippleMergeAnimation (line 1636) is a genuinely nice 1400ms multi-interval animation (position/scale Interval(0,0.65, easeInOut), ripple Interval(0.6,1.0, easeOut), scale 0.5→2.5, blur 16 shadow) — this is the bestcrafted motion in the trade flow. 

- **P2P** (p2p/*): _CashBalanceCard is deliberately theme-independent — gold 0xFFD4AF37 chip, gradient 0xFF0E1116→0xFF05070A, diagonal sheen 3600ms. Two documented fixes: 2026-07-06 (hardcode brand palette so it doesn't shift with theme) and 2026-07-08 (decorative gradient layer collapsed to zero height in an unbounded scroll Column; fixed with Positioned.fill). PeerTransferCard is 240px wide with 5 skins (classic/gold/midnight/emerald/sunset) and an entrance pop easeOutBack 320ms. P2P leaderboard hardcodes 0xFFD4AF37/0xFFB87333. 

- **Wallet pass** (wallet_pass_screen): _addToGoogleWallet only shows a snackbar ("In production: launch URL"); the Apple path writes an **unsigned pass.json** , not a real .pkpass. This surface is a prototype. 

- **Auction** (azm_auction_screen): _Hero hardcodes 0xFF6C63FF/0xFF00E5FF/0xFF9D8FFF; _BidForm uses a raw DropdownButton. 

# **3. Premium micro-interactions that already exist** 

|Widget|Quality in code|
|---|---|
|slide_to_confirm.dart|**Genuinely good.**Thumb 56px, height<br>60, radius 30, threshold**0.90**, reset<br>300ms Curves.elasticOut, gradient track<br>fill, enabled prop (Opacity 0.6), self-<br>healing didUpdateWidget on<br>isLoading/enabled transitions,<br>public reset().|
|azaman_send_button.dart|**Genuinely good and original.**38×38<br>circle, custom 3-part SVG logo<br>rotated math.pi/4, fills<br>biggest→middle→smallest<br>over**480ms**forward /**360ms**reverse,<br>then fires onSend.|
|animated_number.dart|Solid. Tween count-up,<br>respects MediaQuery.disableAnimation<br>s, default MotionTokens.standard.|
|success_celebration.dart|Adequate but generic. 40 confetti<br>particles, **2000ms**,scale|



|Widget|Quality in code|
|---|---|
||600ms elasticOut,<br>palette 0xFFD4AF37/0xFF02C076/0xFF0<br>0E5FF/0xFFBB86FC/0xFFF0B90B —<br>hardcoded, and the cyan/purple clash<br>with the teal/amber dark theme.|
||**The best-engineered widget in the**<br>**set.**4 variants|
|azaman_button.dart|(primary/secondary/danger/ghost), 3<br>sizes (32/44/52h), press scale**0.96**<br>**@120ms**, one-shot shimmer on primary<br>mount, gradient fills.|
||Good. Radius 20, icon circle 56px,|
|azaman_confirm_sheet.dart|confirm h50, cancel h48, haptic vocab<br>(AzamanHaptics.warn/commit/nav).|
|apple_wallet_sliver.dart|**Real**<br>**engineering.**Custom RenderSliverMulti<br>BoxAdaptor, collapsedHeight 64,<br>expandedHeight 184.|
|chat_money_card.dart / _CashBalanceCard|Strong brand identity, intentionally<br>theme-independent.|
|peer_transfer_card.dart|Good, 5 skins, easeOutBack 320ms<br>entrance.|
|escrow_status_panel.dart, loyalty_stamp_card.dart,<br>al_currency_text.dart, vault_progress_card.dart|du<br>Competent, consistent with the vault<br>family.|



**The problem is not that these are bad — it's that they are barely used.** AzamanButton exists but active_trade_screen uses raw ElevatedButton (lines 1417, 1555, 1585); AzamanConfirmSheet exists but most screens use raw showDialog<bool>. The premium primitives are islands. 

# **4. Concrete weaknesses** 

**Competing confirmation patterns — at least 6 distinct idioms for "are you sure?":** 

1. AzamanConfirmSheet bottom sheet (escrow) 

2. Raw showDialog<bool> AlertDialog — vault_detail._confirmBreak, trade_accounts._con firmDelete, deposit_screen._validateAndConfirm (line 531) 

3. SlideToConfirm — withdrawal, upload_proof 

4. **No confirmation at all** — send_money_screen sends immediately 

5. showModalBottomSheet rules-consent — vault_create 

6. Plain ElevatedButton with no gate — vault create, susu config, trade accounts add 

# **Hardcoded colors (not theme** 

**tokens):** p2p _CashBalanceCard 0xFFD4AF37/0xFF0E1116/0xFF05070A (documented as intentional); azm_auction._Hero 0xFF6C63FF/0xFF00E5FF/0xFF9D8FFF; vault_yield 0xFF1B5E20 /0xFF2E7D32; success_celebration + susu_completion confetti 

palettes; credit_score tiers 0xFF00D97E/0xFF00B4D8/0xFFFF9500/0xFFFF3B30; p2p leaderboard 0xFFD4AF37/0xFFB87333; _AzmProgressBar 0xFFD4AF37; spending_insights categ ory colorValues (0xFFEF4444 etc.); chat_money_card skin palette. 

**Inconsistent radii:** 4, 5, 6, 8, 9, 10, 11, 12, 14, 16, 18, 20, 22, 24, 26, 28, 30, 999 all appear. Within a single screen: withdrawal uses 16 (amount field), 20 (Add account pill), 12 (buttons), 14 (Smart Routes card), 2 (toggle underline). Deposit uses 22 (chips/pill), 12, 14, 10. There is no radius scale. 

**Stock Material leakage:** TabBar/TabController (deposit, 

trades_tab); DropdownButton (azm_auction _BidForm); ListTile (susu_completion _MemberPay outRow, 

susu_position_picker _MemberRow); ExpansionTile (azm_rewards); ChoiceChip (vault_create, susu_config); CheckboxListTile/SwitchListTile/Switch.adaptive (several); showDatePicker (vault_ create, shared_vault, susu_config); AlertDialog; IntrinsicHeight. 

**Missing states:** send_money has no loading skeleton (inline only); withdrawal has no success screen (pops); several screens use bare Center(child: CircularProgressIndicator()) (withdrawal line 758, deposit line 297); vault_list._ErrorView uses 

raw Icons.cloud_outlined + Colors.redAccent instead of theme tokens. 

**Dead / placeholder code:** spending_insights._buildBudgetGoals is a "Coming Soon" card (BudgetGoal model defined, unused); wallet_pass_screen._addToGoogleWallet is a snackbar stub; shared_vault "Vault Chat" button has an empty onPressed; susu_position_picker dead trig helpers; _RecipientAvatar uses Icons.send_rounded as a row trailing affordance; susu_credit_score sparkline is fake noise. 

**Potential inconsistency to flag:** vault_create._RulesSheet states a **5% early-break penalty** in the rules text, but vault_detail._confirmBreak reads v.earlyBreakPenaltyPct * 100 dynamically. If the backend value differs from 5, the rules text and the dialog disagree. **[UNVERIFIED — I did not read the backend value.]** 

# **5. Honest verdict** 

# **Would already read as premium fintech:** 

- vault_list / vault_detail / vault_yield — coherent backdrop, BackdropFilter cards, real progress semantics, consistent type scale. 

- active_trade_screen — despite the raw buttons, the timer header, dispute state machine, _RippleMergeAnimation and escrow panel give it a real product feel. 

- p2p_marketplace_screen — the _CashBalanceCard and flip-card transition are distinctive and confident. 

- azaman_button.dart, slide_to_confirm.dart, azaman_send_button.dart, apple_wallet_sli ver.dart — these are genuinely well-built primitives. 

# **Would read as prototype:** 

- send_money_screen — no confirmation, no success moment, no loading skeleton. The single most important flow in a money app is the least finished. 

- wallet_pass_screen — stub Google Wallet, unsigned Apple pass. 

- susu_credit_score — simulated score, fake sparkline. 

- spending_insights — "Coming Soon" budget card. 

- azm_auction_screen — hardcoded purple/cyan hero + DropdownButton bid form. 

- susu_completion / susu_position_picker — raw ListTile rows. 

**The core diagnosis:** the app has a strong _component_ library and a 

weak _system_ . AzamanButton, AzamanConfirmSheet, SlideToConfirm and SuccessCelebration are all good enough to ship — but they are used inconsistently, so the same user action ("confirm this payment") looks like four different products depending on which screen they're on. The highest-leverage fixes are (a) one confirmation idiom, (b) one success idiom, (c) a radius/type scale, and (d) replacing the hardcoded palettes 

in azm_auction, vault_yield, credit_score and success_celebration with theme tokens. 

Confirmed: zero Semantics/Tooltip usage in the home shell widgets. I now have everything needed for the audit. 

# **Azaman Flutter UI Audit — Shell, Navigation, Entry, HOME** 

Read-only. All citations are file:line. Values are exact as written in source. Anything I could not verify is flagged **[UNVERIFIED]** . 

# **1. App shell** 

**Tab structure.** Four tabs, hardcoded in main.dart:230-235 and main.dart:250-261: 

|id<br>x|label|screen|source|
|---|---|---|---|
|0|Home|AzamanHomePage|main.dart:231|
|1|Chat|FriendsHubScreen|main.dart:253|
|2|P2P|P2PMarketplaceScreen|main.dart:255|



- 3 MarketMarketplaceHomeScreenmain.dart:257 

_pages is a List<Widget?> seeded with only index 0 non-null (main.dart:230-235); tabs 1-3 are lazily instantiated on first selection (main.dart:265) and then cached. Once built, all visited tabs stay alive in an Offstage stack (main.dart:441-446) — so state is preserved but **every visited tab's widget tree, providers, and timers stay mounted forever** . There is 

no AutomaticKeepAlive/dispose policy; a user who visits all four tabs holds four full screens in memory simultaneously. 

**Bottom nav.** PremiumBottomNav (premium_bottom_nav.dart:33) is a floating pill: height: 62 (:51), borderRadius: 31 (:54), color: colors.surface (:53), 

shadow Colors.black alpha 0.45 dark / 0.13 light, blurRadius: 24, offset (0,6) (:57-59). Outer padding fromLTRB(16, 0, 16, bottom>0 ? bottom+8 : 16) (:49). It is set 

as Scaffold.bottomNavigationBar with extendBody: true (main.dart:431-432), so content scrolls under it. 

Icons are HugeIcons stroke/solid pairs (:27-30): home01, message01, creditCard, store01. Selected color colors.accent, unselected colors.textTertiary (:112). Label fontSize: 10, weight w700 selected / w500 unselected, letterSpacing: -0.2 (:135-139). 

**Badges** (:165-196): tab 1 = numeric unread chat count; tab 2 = dot for active trades; tab 3 = dot for marketplace notifications. Badge color is hardcoded Color(0xFFEF4444) (:223, :237) — bypasses the theme's colors.danger. Numeric badge fontSize: 9, w800 (:228). 

**Tab-switch animation.** A single AnimationController _fadeCtrl with duration: MotionTokens.standard = **220ms** (main.dart:237, motion_tokens.dart:29). The opacity curve is a hand-rolled V-shape (main.dart:281-285): fades to 0 by v=0.25, then back to 1 by v=1.0. The displayed index swaps at v>=0.25 via a listener (main.dart:238-242). So the outgoing tab fades out over ~55ms and the incoming fades in over ~165ms — an asymmetric crossfade, not a slide. Reduced-motion path sets _displayedIndex instantly and skips the controller (main.dart:266278). 

**Liquid indicator.** LiquidTabBackdrop (liquid_tab_backdrop.dart:9) is a TweenAnimationBuilder keyed on selectedIndex (:46), **420ms** , Curves.easeOutCubic (:48-49). It computes a squash-and-stretch: stretch = 1 + 0.35*(1-|2t-1|), squash = 1 - 0.12*(1-| 2t-1|) (:53-54), rendering a 40px blob at color.withValues(alpha: 0.14) (:69), radius 20*squash (:70). Note the blob's own position uses Curves.easeOutCubic.transform(t) applied _again_ on top of the builder's alreadycurved t (:59) — a double-ease that makes the travel feel slightly frontloaded. **[UNVERIFIED]** whether this is intentional. 

**Backdrop/gradient system.** Two near-duplicate implementations: 

- ThemedAppBackdrop (themed_app_backdrop.dart:24) wraps the whole MaterialApp.router output (main.dart:205-207). Base LinearGradient topLeft→bottomRight 

from colors.background to alphaBlend(accent @ 0.06 dark / 0.030 light) (:36-46). Layer 1: radial glow top-left Alignment(-0.85,-0.95), radius 1.4, colors.glow @ 0.20/0.06 dark, 0.10/0.04 light, stops [0, 0.35, 1] (:55-64). Layer 2: radial accentSecondary bottom-right Alignment(0.95,1.0), radius 1.2, 0.12 dark / 0.06 light (:72-80). Layer 3 (light only): center wash colors.surface @ 0.80 (:85-100). 

- ThemedScaffold (themed_scaffold.dart:32) re-implements the _same_ three layers with slightly different alphas (glow 0.18/0.06 dark, 0.10/0.04 light at :8788; accentSecondary 0.10 dark at :104; center wash 0.95 at :121) and a base gradient at 0.04 dark / 0.025 light (:146). **Two sources of truth for the same visual** — a maintenance hazard and a subtle inconsistency (0.20 vs 0.18 glow, 0.12 vs 0.10 secondary, 0.80 vs 0.95 wash). 

AzamanHomePage itself uses a bare Scaffold(backgroundColor: Colors.transparent) (home_screen.dart:69-70), so it relies on the app-level backdrop. 

# **2. HOME screen composition** 

AzamanHomePage (home_screen.dart:37) is 

a SingleChildScrollView with AlwaysScrollableScrollPhysics(parent: ClampingScrollPhysics()) and padding: EdgeInsets.only(bottom: 120) (:77-81). Top-to-bottom, in order: 

|#Section|Source|Visual treatment|
|---|---|---|
|1SizedBox(height: 8)|:85|—|
|2_GreetingHeader|:87,<br>def :122|Row: 46px gradient-ring avatar (Hero profile-avatar) + Spacer<br>+ AZM rewards glass pill + NotificationBell + visibility toggle.|
|3_GreetingTitle|:91,<br>def :303|<sup>fontSize: 27, w800, letterSpacing: -0.5 (:318-323).</sup>|
|4_ActionPills|:95,<br>def :340|Horizontal scroll of 4 pills: Add Money / Send / Withdraw /<br>History.|
|5_BalanceCardsScroll|:99,<br>def :422|<sup>SizedBox(height: 180) horizontal ListView of 3 cards.</sup>|
|6_SusuShortcutCard|:103,<br>def :571|Glass row w/ 56px progress<br>ring;**renders SizedBox.shrink() unless an active susu**<br>**exists**(:582).|
|7RecentActivitySection|ii:107|Section header + up to N transaction rows.|
|8LiveMarketSection|:111|"TODAY'S RATE" + USDC→GHS hero card + sparkline + rate-<br>alert row.|



**Scroll depth.** Rough vertical budget at 1x: 8 + ~46 (header) + 16 + ~34 (title) + 16 + ~70 (pills) + 18 + 180 (cards) + 28 + ~92 (susu) + 28 + ~(header 34 + 14 + 3×~62 rows ≈ 234) + 28 + ~(header 20 + 10 + hero ~200 + alert ~50 ≈ 280) + 120 bottom pad ≈ **1,150–1,300 logical px** on a populated account. On a 6.1" phone (~800px viewport) that is **~1.5–1.7 screens of scroll** . With an empty account (no susu, no txns) it collapses to ~700px. **[UNVERIFIED]** exact row count in RecentActivitySection — it renders txns.length rows unbounded (recent_activity_section.dart:87), so depth scales with the summary payload. 

**Competing modules stacked.** Eight distinct modules, of which **five are "money/status" surfaces competing for the same glance** : the balance card carousel (3 cards), the susu progress card, recent activity, and the live-rate hero. The balance carousel alone stacks three different card idioms side by side (home_screen.dart:436-454): 

- FlippableBalanceCard at screenWidth * 0.76 (:438) 

- _NewWalletCard at screenWidth * 0.38 (:446) 

- _MarketplaceShortcutCard at screenWidth * 0.38 (:451) 

**Card geometry is inconsistent across the home screen.** Radii in use: 22 (flippable/hologram back 

face flippable_balance_card.dart:270, hologram_balance_card.dart:32, _NewWalletCard home_ screen.dart:475, _MarketplaceShortcutCard :530), 20 (susu card :597, dashboard card dashboard_balance_card.dart:131), 16 (live-market hero live_market_section.dart:165, onboarding use-case onboarding_screen.dart:471), 14 (today 

tiles today_widget.dart:258), 12 (settings tiles settings_drawer.dart:190). Heights: balance carousel fixed 180 (:431) vs HologramBalanceCard minHeight: 158 (hologram_balance_card.dart:29) vs DashboardBalanceCard aspectratio 1.586 (dashboard_balance_card.dart:49,81). **Three different balance-card components exist** (dashboard_balance_card.dart, hologram_balance_card.dart, flippable_balance_card.dart) and the home screen uses only the flippable one. 

**Dead code on home.** TodayWidget (today_widget.dart:48) 

and QuickActionsRow (quick_actions_row.dart:11) are **not referenced by home_screen.dart** — verified by grep; TodayWidget is only mentioned in a comment 

in home_summary_service.dart:18. QuickActionsRow is referenced nowhere outside its own file. StoryRing is used in contacts/messages/friends/marketplace but **not on home** . So the home screen has three orphaned "home" widgets and no story rail, despite the app shipping a full story system. 

# **3. Entry experience** 

**Splash** (splash_screen.dart:24). Scaffold(backgroundColor: Colors.white) (:182) — **hardcoded white, ignores theme entirely** , so a dark-theme user gets a white flash. Content: LogoTraceLoader(size: 120, strokeWidth: 4, color: Colors.black @ 0.75, loopDurationSeconds: 2.4) (:193-198) + "AZAMAN" wordmark fontSize: 22, w700, letterSpacing: 8 (:216-221) with a ShaderMask vertical black gradient (:201-213) and a repeating 

shimmer 2200ms (:224-225). Entrance: AnimationController **800ms** (:45), fade Curves.easeOut (:47), scale 0.85→1.0 Curves.easeOutCubic (:48-50). 

Then _checkAuthStatus (:69) does await Future.delayed(Duration(seconds: 2)) (:70) — a **hard 2- second floor** before any routing decision, regardless of how fast auth resolves. On web it additionally probes /health with a 3s timeout and silently flips to demo mode (:73-82). Routing is a chain of Navigator.pushReplacement(MaterialPageRoute(...)) (:101, 111, 145, 147, 151, 159, 163, 166, 169) — **not GoRouter** , so the splash→login→main path bypasses the router's transition system entirely and uses the default MaterialPageRoute (which is overridden by AzamanPageTransitionsBuilder via theme, so it does get the shared-axis slide — see §4). 

**Onboarding** (onboarding_screen.dart:27). 4 pages: 3 intro + 1 setup (:40). Intro pages use assets/images/1.webp, 2.webp, 3.webp (:45, 52, 60) in a ClipRRect(radius 24) Image.asset (:294-301). Top: LinearProgressIndicator minHeight: 3, radius 3, colors.accent (:133-140) + "Skip" TextButton (:145). Bottom: full-width ElevatedButton height: 52, radius 14, colors.accent (:204-219). Page transition: PageController.nextPage(duration: 400ms, curve: Curves.easeInOutCubic) (:105-108). Dots: AnimatedContainer **300ms** Curves.easeInOut, active width 24 vs 8, height 8, radius 4, with a glow shadow blurRadius 8 (:555-573). Setup page: 4 use-case cards, AnimatedContainer **200ms** (:465), selected border 1.5 vs 1 (:476), AnimatedSwitcher **200ms** for the check icon (:518-523). On completion it pushAndRemoveUntil to MainNavigationWrapper then, after **700ms** , forcepushes DepositScreen (:92-100) — an aggressive post-onboarding deposit prompt. 

**Login** (login_screen.dart:20). Two-step flow (_step 0/1, :42). Step 0: Hero azaman_logo (:267) with a TweenAnimationBuilder shrinking 100→80 over **600ms** Curves.easeOutCubic (:269-272); "Welcome to Azaman" fontSize: 28 bold (:288-292); email + password TextFields with fillColor: colors.card, radius 12, BorderSide.none, focused border accent @ 0.5 (:322-336); Login button height: 52, radius 12 (:369-380); "or continue with" divider; Apple (black) + Google (white) SSO buttons height: 50, radius 12 (:757-772); Sign Up link; **"Try Demo (No Account Needed)"** OutlinedButton (:474-496); influencer-code link (:513-524). Step 1 is an influencercode screen (:529). Step transition: _fadeController **500ms** Curves.easeInOut (:47-51), reverse→swap→forward (:68-80). SSO is **not actually wired** — it shows an AlertDialog explaining Firebase isn't compiled in (:703-745). 

**Signup** (signup_screen.dart:16). Standard AppBar "Create Account" (:185- 

193), Icons.swap_horiz at size: 64 as the hero icon (:201) — **the same generic swap icon used for the P2P tab** , a weak brand moment. Password strength bar with hardcoded colors 0xFFEF4444 / 0xFFF59E0B / 0xFF10B981 / 0xFFE5E7EB (:606-618). Same SSO-notconfigured dialog (:496-538). 

# **4. Motion inventory** 

**Tokens** (motion_tokens.dart): instantFeedback 90ms, microInteraction/fast 120ms, control 180ms, standard 220ms, emphasized 350ms, spatial 450ms, ambient 900ms, celebration 1200ms (:17-41). Curves: enter easeOutCubic, exit easeInCubic, spring easeOutBack, symmetric easeInOutCubic, decelerate (:45-61). Stagger 40ms step, 300ms cap (:65-68). 

**Router transitions** (transitions.dart): horizontalPage 300ms fwd / 250ms rev (:2728); fadeThroughPage 250/200 (:51-52); verticalPage 350/300 (:75-76); legacy sharedAxisPage 300/300 (:99-100). All bail to child when MediaQuery.disableAnimations (:30, 54, 78, 102). 

**Global page builder** (azaman_page_transitions.dart:22): inbound slide Offset(0.06,0)→0 + fade, outbound 0→Offset(-0.04,0) + dim 1.0→0.92, curves MotionTokens.enter/exit (:38-61). Reduced-motion → crossfade only (:34-36). 

**Nav helpers** (nav_transitions.dart): _SharedAxisRoute uses MotionTokens.standard (220ms) fwd / fast (120ms) rev (:78-79), Curves.easeOutCubic (:85). Vertical slide Offset(0,0.06) (:98), horizontal Offset(0.06,0) (:112), scaled 0.92→1.0 (:125). 

**Home screen** (home_screen.dart), all flutter_animate: 

- Section entrance stagger: header fadeIn 320ms (:87); title fadeIn 360ms + slideY(0.1) delay 100ms (:91); pills fadeIn 350ms + slideY(0.15) delay 200ms (:95); balance cards fadeIn 400ms + slideY(0.15) delay 300ms (:99); susu fadeIn 400ms + slideY(0.1) delay 400ms (:103); recent activity delay 500ms (:107); live market delay 600ms (:111). **Total entrance cascade ≈ 1,000ms.** 

- Avatar: fadeIn 300ms + scale(0.8→1.0) 300ms Curves.easeOutBack (:201-206). 

- AZM gift icon: **infinite** repeat(reverse: true) scale 1.0→1.1 800ms easeInOut (:227-233). 

- Visibility toggle: AnimatedSwitcher 250ms with RotationTransition (0.5→1.0 turns) + ScaleTransition (:267-274). 

- Action pills: per-pill fadeIn 300ms + slideY(0.2) staggered i*80ms (:367-375); each pill icon has an **infinite** shimmer 2000ms (:402-406). 

- _NewWalletCard: **infinite** shimmer 3000ms (:502-506). 

- _MarketplaceShortcutCard: fadeIn 400ms + slideX(0.2) delay 200ms (:565-567). 

- Susu ring: scale(0.5→1.0) 400ms easeOutBack (:627-628); card fadeIn 400ms + slideY(0.1) delay 300ms (:654-656). 

**Balance cards** : FlippableBalanceCard flip AnimationController 520ms Curves.easeInOutCubic (flippable_balance_card.dart:59-63), rotateX with perspective setEntry(3,2,0.001) (:164-166). HologramBalanceCard entrance fadeIn 380ms + slideY(0.04) (hologram_balance_card.dart:174-178); AnimatedNumber **600ms** (:117). DashboardBalanceCard titanium sweep **6s infinite** (dashboard_balance_card.dart:115-118); live pulse dot **2s infinite reverse** (:640-643); oracle countdown Timer.periodic(1s) (:530). 

**Nav** : tab crossfade 220ms (main.dart:237); liquid blob 420ms (liquid_tab_backdrop.dart:48); icon TweenAnimationBuilder MotionTokens.fast (120ms) easeOutBack (premium_bottom_nav.d art:123-126); label AnimatedDefaultTextStyle 120ms easeOut (:132-134); icon swap AnimatedSwitcher 120ms (:151-155). 

**Sections** : RecentActivitySection header fadeIn 300ms delay 100ms (:78); rows fadeIn 280ms + slideY(0.08) staggered i*80ms (:89-91). LiveMarketSection has **no entrance animation** — it appears instantly, inconsistent with every sibling. TodayWidget count AnimatedSwitcher 280ms with slide Offset(0,0.15) (today_widget.dart:288-299); header spinner AnimatedSwitcher 250ms (:210-211). NotificationBell fadeIn 300ms (notification_bell.dart:92) + badge TweenAnimationBuilder 800ms easeInOut **infinite reverse** (:48-51, 87). 

**Entry** : splash 800ms (splash_screen.dart:45), logo trace loop 2.4s (:197), shimmer 2200ms (:225); onboarding page 400ms (onboarding_screen.dart:106), dots 300ms (:556), use-case 200ms (:465), check 200ms (:519); login step 500ms (login_screen.dart:49), logo morph 600ms (:271). 

**Reduced-motion coverage is partial.** Respected in: router transitions (transitions.dart:30,54,78,102), global page builder (azaman_page_transitions.dart:34), nav helpers (nav_transitions.dart:82), tab crossfade (main.dart:266), nav button scale/label/icon (premium_bottom_nav.dart:113,125,133,150). **Not respected** in: every flutter_animate call on the home screen, the infinite shimmers, the 6s titanium sweep, the 2s pulse dot, the notification-bell pulse, the splash/onboarding/login 

animations. MotionTokens.accessibleDuration exists (motion_tokens.dart:79) but is not used by any of these. 

# **5. Concrete weaknesses** 

**Overloaded home.** Eight modules, five of them money/status surfaces, ~1.5–1.7 screens of scroll, and a 1,000ms entrance cascade before the last section is even visible. The balance carousel puts three unrelated card idioms in one horizontal strip (home_screen.dart:436-454). There is no single dominant "what is my money doing" answer — the user must parse a 3-card carousel, a susu ring, a transaction list, and a rate hero. 

**Inconsistent card geometry.** Radii 12/14/16/20/22 all in use across home + adjacent surfaces (see §2). Three separate balance-card components (dashboard_balance_card.dart, hologram_balance_card.dart, flippable_balance_card.dart) with three different sizing strategies (aspect ratio 1.586, minHeight 158, fixed parent 180). Two duplicate backdrop implementations with divergent alphas (themed_app_backdrop.dart vs themed_scaffold.dart). 

**Hardcoded colors bypassing the theme.** Color(0xFFEF4444) for nav badges (premium_bottom_nav.dart:223,237); Color(0xFF02C076) for success snackbars (login_screen.dart:163, signup_screen.dart:148); Color(0xFFFFD700) story ring (story_ring.dart:46); Color(0xFF161618)/0xFF0B0B0D/0xFF000000 card gradient (dashboard_balance_card.dart:158-160); password-strength palette (signup_screen.dart:606618); Colors.white splash scaffold (splash_screen.dart:182); Colors.black "Add" button text (recent_activity_section.dart:285). None of these respond to theme switching, and several (0xFFEF4444, 0xFF02C076) duplicate semantic roles the theme already defines as danger/success. 

**Dead code.** TodayWidget (today_widget.dart:48) 

and QuickActionsRow (quick_actions_row.dart:11) are unreferenced by the home screen (grepverified). _BalanceNumber in hologram_balance_card.dart:182-284 is a fully-implemented stateful animated-number widget that is **never instantiated** — the card uses AnimatedNumber instead (:112). _RoutedSurface in today_widget.dart:37-46 is a trivial pass-through wrapper around RoutedTabSurface. MainNavigationWrapper (main.dart:458) is a one-line alias for MainWrapper. _publicRoutes in auth_guard.dart:19-22 is declared but never read. 

**Competing design idioms.** Three navigation systems coexist: GoRouter (app_router.dart), raw Navigator.push(MaterialPageRoute(...)) (used throughout home, splash, login, signup, settings drawer), and the custom pushWithVerticalTransition helpers (nav_transitions.dart). The home screen mixes all three: Navigator.push for profile/rewards (home_screen.dart:144, 211), pushWithVerticalTransition for pills (:348-354), and context.push for susu (:592). Splash and auth use MaterialPageRoute exclusively, bypassing GoRouter. app_router.dart:1-22 explicitly documents this as intentional, but the result is that deep-link and in-app navigation have different transition and back-stack semantics. 

**Reads as stock Material.** RoutedTabSurface (routed_tab_surface.dart:29-47) is a default AppBar + Icons.arrow_back at size: 18. SignUpScreen uses a stock AppBar (signup_screen.dart:185) and Icons.swap_horiz at 64px as its hero (:201). Login/signup TextFields are 

default OutlineInputBorder with BorderSide.none (login_screen.dart:328-331). The SSO-notconfigured AlertDialog (login_screen.dart:709) is unstyled Material. _DisputeScreen (app_router.dart:728-757) is a bare AppBar + centered icon. 

**Performance risks.** 

- **Unbounded rebuild** 

**surface.** AzamanHomePage.build watches themeProvider (home_screen.dart:67) — the whole page rebuilds on any theme notify. _GreetingHeader watches authProvider (:128) and balanceVisibleProvider (:129); _GreetingTitle watches authProvider (:309). Hologra mBalanceCard watches balanceDataProvider + authProvider (hologram_balance_card.da rt:19-20) and nests a Consumer for rate/currency 

(:93). LiveMarketSection watches homeSummaryProvider + rateHistoryProvider (live_ma rket_section.dart:69-71). RecentActivitySection watches homeSummaryProvider (recent _activity_section.dart:37). Because homeSummaryProvider is a single StateNotifier holding the entire snapshot (home_summary_provider.dart:24), **any section's data change rebuilds every consumer of it** — there is no select() partitioning on the home screen (contrast with dashboard_balance_card.dart:76, 298, 406 which does use select() correctly). 

 **Infinite animations always running.** At least five repeat() animations are live whenever home is mounted: AZM gift pulse (home_screen.dart:227), 4× pill shimmer (:402), wallet-card shimmer (:502), notification-bell pulse (notification_bell.dart:87), plus the 6s titanium sweep and 2s pulse dot if DashboardBalanceCard is mounted. None pause when the tab is Offstage (main.dart:443) — Offstage stops painting but **does not stop tickers** , so a user on the Chat tab still burns CPU on home's shimmers. 

 **Large single-file widgets.** settings_drawer.dart is **1,264 lines** ; login_screen.dart 794; home_screen.dart 661; signup_screen.dart 627; onboardin g_screen.dart 576; live_market_section.dart 509; today_widget.dart 481. _GreetingHea der alone is ~180 lines of inline layout (home_screen.dart:122-301). 

- **Timer.periodic in _OracleRefreshPill** ticks every second and calls setState (dashboard_balance_card.dart:530-536) — a 1Hz rebuild of that subtree for a countdown that only needs to update once per second at most, and it runs even when the card is off-screen. 

- **oracleRateProvider** starts a Timer.periodic(60s) and a Future.microtask(refresh) at provider construction (hologram_provider.dart:201-203) — a network poll that runs for the app's lifetime once anything reads it. 

# **Accessibility gaps.** 

- **Zero Semantics, semanticLabel, Tooltip, or ExcludeSemantics** in any of the audited shell/home widgets (grep-verified 

across home_screen.dart, premium_bottom_nav.dart, notification_bell.dart, recent_acti vity_section.dart, live_market_section.dart, flippable_balance_card.dart, hologram_bala nce_card.dart). Icon-only controls — the visibility toggle (home_screen.dart:253-285), the notification bell (notification_bell.dart:26), the nav icons 

(premium_bottom_nav.dart:116) — have no accessible labels. 

- **Touch targets below 48dp.** Nav icons are size: 22 inside a 62px bar split 4 ways (~90px wide, so width is fine, but the tappable Column is icon+4+label ≈ 40px tall). The visibility toggle is 40×40 (home_screen.dart:264-265). The notification bell is 40×40 (notification_bell.dart:33). The balance-card eye icon is size: 13 with EdgeInsets.symmetric(horizontal: 4, vertical: 

   - 2) (dashboard_balance_card.dart:432-439) — roughly 21×17px. The _BalanceLine icon is 22×22 (flippable_balance_card.dart:368-369). 

- **Contrast risks.** colors.textTertiary is used for 9–11px labels throughout (home_screen.dart:412, live_market_section.dart:86, today_widget.dart:318). Colors.wh ite.withValues(alpha: 0.45) for the UID line at fontSize: 

9.5 (dashboard_balance_card.dart:328) and alpha: 0.50 for the "TOTAL PORTFOLIO VALUE" label at 9.5 (:418) are almost certainly below 4.5:1 on the near-black card. **[UNVERIFIED]** without a contrast measurement, but the alpha values are low enough to flag. 

- **Reduced-motion** is only partially honored (see §4) — the infinite shimmers and pulses ignore disableAnimations entirely. 

- **_ActionPills** is a horizontal SingleChildScrollView (home_screen.dart:356) with no scroll affordance or semantics hint that more content exists off-screen. 

# **6. Verdict vs. Phantom / Revolut** 

**Structure.** Phantom and Revolut both lead with a single, unambiguous primary surface: one balance, one primary action set, and a short, scannable feed. Azaman's home instead presents **eight modules and three balance-card components** , with the balance itself split across 

a flippable card, a wallet card, a marketplace card, a susu ring, and a rate hero. The information architecture is additive rather than curated — it reads like every sprint appended a section without removing one. The presence of two orphaned home widgets (TodayWidget, QuickActionsRow) and an unused animated-number widget confirms that sections were built and then abandoned rather than consolidated. 

**Restraint.** This is the biggest gap. Phantom's motion is sparse and purposeful; Azaman runs **at least five infinite animations on the home screen alone** (gift pulse, 4× pill shimmer, wallet shimmer, bell pulse, plus the card sweep and pulse dot), and its entrance cascade takes a full second. The flutter_animate shimmer on every action pill (home_screen.dart:402-406) is decorative noise with no informational value. The 6-second titanium sweep and 1Hz oracle countdown (dashboard_balance_card.dart:115, 530) are ambient effects that a top-tier app would cut or gate behind reduced-motion. 

**Craft.** The motion token system (motion_tokens.dart) is genuinely good and better than most production Flutter apps — but it is **only partially adopted** : the home screen hardcodes durations (320ms, 360ms, 350ms, 400ms, 500ms, 600ms) instead of pulling from tokens, and reduced-motion is honored in the router but ignored in the widgets. The theme system (AzamanColors) is well-structured, but hardcoded hex values leak through in nav badges, snackbars, the story ring, the balance card gradient, and the splash screen. The backdrop system is duplicated across two files with divergent values. 

**Where it's competitive.** The LiquidTabBackdrop squash-and-stretch (liquid_tab_backdrop.dart:53-54) is a genuinely nice, cheap touch that reads as more considered than a stock NavigationBar. The FlippableBalanceCard flip-to-breakdown (flippable_balance_card.dart:144-181) is a real product idea — surfacing escrow/vault/susu/savings locked balances on the back of the card is the kind of thing Revolut does well. The select()-based granular rebuilds in dashboard_balance_card.dart show the team knows how to do this correctly. 

**Honest summary.** The _ingredients_ are close to top-tier — good motion tokens, a real theme system, thoughtful individual widgets. The _composition_ is not. Azaman's home screen is roughly 1.5–1.7 screens of stacked, competing modules with inconsistent card geometry, five always-on animations, partial reduced-motion support, and no accessibility semantics on any icon-only control. Against Phantom or Revolut, the gap is not visual polish — it is **editorial discipline** : those apps ship fewer surfaces, each with one clear job, and they cut motion that doesn't carry information. Azaman has the opposite problem: it ships every surface and animates all of them. 

**Highest-leverage fixes, in order:** (1) collapse the three balance-card components into one and remove the two orphaned home widgets; (2) gate all infinite animations behind MediaQuery.disableAnimations and delete the decorative pill shimmer; (3) 

add Semantics labels to the icon-only controls and raise sub-48dp targets; (4) replace hardcoded hex with theme tokens; (5) partition homeSummaryProvider with select() so a rate tick doesn't rebuild the transaction list. 

# **Azaman Chat / Messaging / Social — Read-Only UI Audit** 

**Scope verified:** all listed files read in full. public_profile_modal.dart is at lib\widgets\ (not lib\screens\ as the brief stated) and is **dead code** (see §6). chat_interface.dart is the _trade_ chat (buyer/vendor/admin), not the friend chat — it has its own embedded _PremiumChatInput (line 998) that duplicates premium_chat_input.dart. 

# **1. Chat experience end-to-end** 

# **Conversation list — two competing implementations** 

- **messages_hub_screen.dart** (1001 lines): title "Messages" 27px w800 ls-0.5 (:373); search pill radius 14 (:412); 6 folder chips (All/Unread/Groups/Business/Archived/Favorites) radius 18, active = accent@15% fill + accent@40% border, AnimatedContainer 200ms (:466-478); story rail height 96 (:559); rows: 48×48 radius-14 initial tile, name 15.5px w700, preview 13px, unread pill radius 10 with 9+ cap (:1112), row radius 16 with 0.7px divider border (:1019). Swipe right = mark read (success@85% bg), swipe left = action sheet (:938-995). 

- **friends_hub_screen.dart** (1439 lines): title "Inbox" 28px w800 (:373); **no search bar until toggled** ; groups+friends merged into one recency-sorted list with a gradient hairline separator (:843-857); rows use ChatAvatar 50px squircle + online dot, rating/transaction subtitle, verified check. Empty state uses PremiumGlassContainer blur 20 with a 2000ms breathing scale (:772-773). 

# **These two hubs are near-duplicates with different titles, different row geometry (48px radius-14 tile vs 50px squircle avatar), different empty states, and different swipe** 

**models.** That is the single biggest structural inconsistency in the messaging layer. 

# **Thread** 

- **friend_chat_screen.dart** : reverse: true ListView, PremiumMessageBubble, date headers, TypingBubble, PremiumChatInput. Optimistic send with clientNonce dedupe and 100ms SLA via jumpTo(0) (:301-307). AppBar title = avatar 38 + username 16px w700 + trust line 11px (rating star + "N Completed Transactions"), AnimatedSwitcher 220ms to "typing…" (:855-869). 

- **group_chat_screen.dart** : same bubble/input, showAvatar: true, showSenderName: true, @-mention picker (max 5, radius 12), Susu banner + Susu event cards (emoji pill radius 20). 

- **chat_interface.dart** (trade): a _completely different_ bubble system — _chatBubble radius 16/16/16/0, maxWidth 75%, padding 10, border on incoming only (:896-911); plus milestone/admin/system/urgency/offline/time-extension card variants. Uses BouncingScrollPhysics + keyboardDismissBehavior.onDrag (:184-188). 

# **Composer** 

- **premium_chat_input.dart** : floating pill **height 62, radius 31** , surface fill, shadow blur 24 offset (0,6) (:200-210); inner TextField radius 20, fill softSurface, maxHeight 100; + = LiquidDropdownMenu size 36; send/mic swap via AnimatedSwitcher 180ms ScaleTransition (:286-289); reply strip AnimatedSize 200ms; upload LinearProgressIndicator minHeight 2. 

- **chat_interface.dart _PremiumChatInput** : **flat bar, not a pill** — Container with top border, radius-20 field, 38px send circle (:1060-1155). Explicitly a different visual language from the friend composer. 

# **Attachments / money / stickers / media** 

- **chat_plus_menu.dart** : iMessage-style vertical bubble stack in a real OverlayEntry with full-screen scrim black@16% (:117); 320ms controller, per-item Interval + easeOutBack, translate 24px + scale 0.6→1.0 (:233-244); 42px circles with BackdropFilter blur 6 (:278). **Hardcodes ThemeProvider.getColors(AzamanTheme.light)** (:99,170) — ignores active theme. 

- **chat_money_card.dart** : bank-card bubble, width 74% screen, radius 22, gold #D4AF37 on #0E1116→#05070A gradient, 3600ms sheen (:139), 5 skins with perskin flutter_animate loops (shimmer 2000/2500/3000ms, tint 4000ms). Amount 24px w800 ls-0.5. **Fixed palette, theme-independent by design** (:87). 

- **chat_transfer_sheet.dart** : full keypad sheet, card aspect 1.586, radius 22, amount 48px w800 ls-1.5, 3200ms reverse sheen. **Dead code** (see §6). 

- **sticker_sheet.dart** : height 360, radius 24, 2 tabs, 4-col grid, 6 static PNG + 4 Lottie. **Hardcodes light theme** (:52). 

- **chat_media_bubble.dart** : IMAGE 240×240 radius 12; VIDEO 260×160 with play overlay; AUDIO inline player with waveform (bar width clamp 1–4, height = peak/100 × 

maxHeight), 1×/1.5×/2× speed, single-player registry; DOCUMENT radius 12; LINK 16:9 OG card radius 12. 

# **2. Social layer** 

- **Stories creation** : story_camera_screen.dart — 8 ColorFilter.matrix presets, filter carousel 56×56 radius 16, capture button 72→60 on press (AnimatedContainer 200ms), rule-ofthirds grid. **Camera preview is a fake gradient placeholder** (:207-235) — image_picker does the real capture. story_editor_screen.dart — text/sticker/draw/filter, draggable overlays, _DoodlePainter, 8 text colors, brush 1– 12. story_creation_screen.dart — caption + boost + store-link, radius 24 sheet. 

- **Viewer** (story_viewer_screen.dart): custom PageRouteBuilder 320ms in / 260ms out, easeOutCubic, fade + scale 0.92→1.0 (:31-47); segmented progress bars minHeight 3 radius 3; tap zones 35%/65%; long-press pause; reply pill radius 28; boosted = amberAccent. **_advance()/_rewind() have no cross-fade** — story changes are a hard cut. 

- **Highlights** (story_highlights_screen.dart): 3-col grid aspect 0.8, 80px circle tiles, create/delete dialogs. **_viewHighlight() is a no-op stub** (:175-180). 

- **Analytics** (story_analytics_screen.dart): 6 glass stat cards (3-col), per-story cards, top-10 viewers. Hardcoded stat colors #9C59FF/#EF4444/#3B97F7/#10B981/#FFD700 (:138142). 

- **Close friends** (close_friends_screen.dart): search + list, radius-12 search field, fadeIn stagger 50ms×i. **addFriend/removeFriend swallow all errors** (:50,60). 

- **Profiles** : public_profile_modal.dart is **unused** (see §6). group_profile_screen.dart is the richest social surface: avatar cluster (5 bubbles, 30px offset, 48px, 2px surface border), KYC/PoA verification chips, Susu initiation CTA with gradient + dynamic pool math. 

# **3. Notifications** 

- **Hub** (notification_hub_screen.dart): BackdropFilter blur 16 AppBar, 6 tabs (All/Money/Security/Social/Chat/System), tab chips AnimatedContainer 250ms easeOutCubic radius 20, date-grouped list, PremiumGlassContainer tiles blur 8, unread dot 7px with glow, fadeIn+slideX 250ms. Deep-link logic is careful (uses context.push not go, :481-498). 

- **Overlay** (notification_overlay.dart): top sheet 92% height, slide 420ms easeOutQuart, BackdropFilter blur 28, drag-to-dismiss (velocity < -350 or drag < -80), bobbing handle 1200ms, quiet-hours with SharedPreferences, swipe-reveal rows (240ms, ±110px, easeOutCubic). 

- **Push banner** (in_app_push_banner.dart): the **only accessibility-correct** file in scope — Semantics(liveRegion: true), honors MediaQuery.disableAnimations, uses MotionTokens.standard/enter, 4s auto-dismiss, radius 14, shadow blur 16. 

# **4. Calls** 

- **call_screen.dart** : real WebRTC. Pulsing avatar rings (3 rings, 2000ms, size 120→220, opacity 0.4→0), pulse 1500ms reverse, avatar 130px gradient #6C5CE7→#4834D4 glow blur 30. Controls 60px (end/accept 72px). **Video PiP is a "You" text placeholder** (:386389); **RTCVideoRenderer is created inside build()** (:363) — a new renderer every rebuild, a real leak. 

- **incoming_call_overlay.dart** : 55 lines, listens to activeCall, pushes CallScreen. No ringtone, no full-screen intent, no accept/reject on the overlay itself. 

- **call_history_screen.dart** : **stock Material** — AppBar, ListTile, CircleAvatar, FilledButton, Theme.of(context).colorScheme.* , SkeletonBlock. Zero Azaman theming. _showNewCallDialog is a TODO stub that does nothing on "Call" (:134-139). 

# **5. Motion inventory (verified durations/curves)** 

|File|Animation|Duration|Curve|
|---|---|---|---|
|premium_message_bubb<br>le|reply swipe ctrl|300ms|easeOut|
|premium_message_bubb<br>le|reply arrow opacity|80ms|—|
|premium_message_bubb<br>le|bubble AnimatedContaine<br>r|300ms|easeInOut|
|premium_chat_input|reply strip AnimatedSize|200ms|—|
|premium_chat_input|send/mic AnimatedSwitch|180ms|ScaleTransition|



|File|Animation|Duration|Curve|
|---|---|---|---|
||er|||
|chat_plus_menu|open/close ctrl|320ms|easeOutBack<br>(Interval)|
|chat_money_card|sheen ctrl|3600ms|easeInOutSine|
|chat_money_card|gold shimmer / blurXY|2000/1000ms|—|
|chat_money_card|midnight tint|4000ms|—|
|chat_money_card|emerald shimmer|2500ms|—|
|chat_money_card|sunset shimmer|3000ms|—|
|chat_transfer_sheet|sheen ctrl|3200ms reverse|easeInOutSine|
|typing_indicator_bubble|dot ctrl|900ms|sine (manual)|
|message_status_ticks|AnimatedSwitcher|150ms|Fade|
|chat_avatar|online dot pulse|1200ms×2|easeInOut|
|chat_avatar|hero shuttle|—|easeOutCubic|
|chat_unread_badge|scale in|300ms|elasticOut|
|reply_preview_bar|AnimatedSize|200ms|easeOutCubic|
|inline_link_preview|fadeIn|300ms|—|
|thinking_orb|orbit ctrl|1400ms|linear|
|story_viewer|route in/out|320/260ms|easeOutCubic|
|story_viewer|pause fadeIn|150ms|—|
|story_viewer|reply button|200ms|—|
|story_camera|capture button|200ms|—|
|story_camera|filter select scale|—|flutter_animate|
|notification_hub|tab chip|250ms|easeOutCubic|
|notification_hub|tile fadeIn+slideX|250ms|—|



|File|Animation|Duration|Curve|
|---|---|---|---|
|notification_hub|empty bell breathe|1500ms|easeInOut|
|notification_overlay|slide in|420ms|easeOutQuart|
|notification_overlay|handle bob|1200ms|easeInOut|
|notification_overlay|swipe reveal|240ms|easeOutCubic|
|in_app_push_banner|slide|MotionTokens.stand<br>d|iar<br>MotionTokens.ent<br>er|
|call_screen|pulse|1500ms reverse|—|
|call_screen|rings|2000ms|—|
|friends_hub|header fadeIn+slideY|300ms|—|
|friends_hub|action buttons|300ms|—|
|friends_hub|story rail stagger|300ms, 80ms×i|—|
|friends_hub|empty glass breathe|2000ms|easeInOut|
|close_friends|list stagger|50ms×i|—|
|story_analytics|stat card fadeIn+scale|100ms×i|—|



**No shared motion token system** except MotionTokens (used only by in_app_push_banner). Every other file hardcodes its own durations/curves. 

# **6. Concrete weaknesses** 

# **Hardcoded colors (theme-blind):** 

- chat_plus_menu.dart:99,170 and sticker_sheet.dart:52 force AzamanTheme.light — these render light-on-dark in dark mode. 

- chat_interface.dart:338,343,348,356,455,492 — #02C076/#EF4444/#FFB800 for timeextension states. 

- chat_money_card.dart:88-153,403,409,471 and chat_transfer_sheet.dart:24-26,76,288 — full fixed palette (intentional for the card, but the status colors #22C55E/#F59E0B/#EF4444 duplicate colors.success/warning/danger). 

- chat_avatar.dart:133 — #02C076 online dot + Colors.grey offline. 

- message_status_ticks.dart:26-30 — Colors.white38/54 + #4FC3F7; **this widget is unused** (see dead code). 

- story_analytics_screen.dart:138-142 — 5 hardcoded stat colors. 

- call_screen.dart:174,316,320,444 — #1A1A2E, #6C5CE7, #4834D4, #2ECC71. 

# **Inconsistent bubble/composer geometry:** 

- Friend/group bubble: radius 18/18/18/4, maxWidth 72%, padding (12,10,12,8), shadow blur 8 (premium_message_bubble.dart:415-432). 

- Trade bubble: radius 16/16/16/0, maxWidth 75%, padding 10, border on incoming (chat_interface.dart:900-911). 

- Friend composer: pill height 62 radius 31 (premium_chat_input.dart:200-203). 

- Trade composer: flat bar, radius-20 field, 38px send (chat_interface.dart:1060-1155). 

- Conversation rows: 48px radius-14 tile (messages_hub:1023) vs 50px squircle ChatAvatar (friends_hub:956). 

- **Four different "send" affordances** : AzamanSendButton, the trade 38px circle, the storyviewer 42px circle, the editor 44px circle. 

# **Stock Material leakage:** 

- call_history_screen.dart — entirely Material (AppBar, ListTile, CircleAvatar, FilledButton, Theme.of). 

- reply_preview_bar.dart — theme.colorScheme.surfaceContainerHighest/primary/ onSurfaceVariant (never uses AzamanColors). 

- inline_link_preview.dart — colorScheme.surfaceContainerHighest/onSurface/ onSurfaceVariant. 

- disappearing_message_timer_sheet.dart — theme.textTheme, cs.surface/primary/ outline. 

- in_app_push_banner.dart — theme.colorScheme.* (acceptable; it's a system-level banner). 

- group_profile_screen.dart:65,69 — bare CircularProgressIndicator() and Text(e.toString()) error state. 

# **Missing states:** 

- story_highlights_screen.dart:175 — _viewHighlight is an empty stub. 

- call_history_screen.dart:134 — "Call" button does nothing. 

- story_creation_screen.dart:85 — video preview is a text placeholder. 

- story_camera_screen.dart:207 — camera preview is a gradient placeholder. 

- call_screen.dart:386 — local video PiP is a "You" text placeholder. 

- messages_hub_screen.dart — no loading skeleton (bare spinner); friends_hub has none either. 

- close_friends_screen.dart:50,60 — silent error swallow on add/remove. 

# **Dead code (verified unused):** 

- chat_transfer_sheet.dart — ChatTransferSheet referenced only in its own file. 

- message_status_ticks.dart — MessageStatusTicks referenced nowhere (bubble has its own _statusTick). 

- reply_preview_bar.dart — ReplyPreviewBar/replyTargetProvider referenced nowhere (composer has its own inline reply strip). 

- thinking_orb.dart — ThinkingOrb referenced nowhere. 

- public_profile_modal.dart — PublicProfileModal referenced nowhere. 

- friend_chat_screen.dart:75 — final bool _isUploadingAudio = false never read; :72 _hasInputText written but never read; :235 _onScroll() never called (a second scroll listener is registered inline at :108). 

# **Performance risks:** 

- call_screen.dart:363 — RTCVideoRenderer() instantiated inside build(); leaks a native renderer per rebuild. 

- chat_money_card.dart:137-140 — a 3600ms repeat() controller runs for **every** money bubble in the list, forever, even off-screen. 

- chat_avatar.dart:146-149 — online-dot pulse repeat() per avatar; a 50-row list = 50 infinite controllers. 

- friends_hub_screen.dart:772 — empty-state glass breathe repeat(reverse: true). 

- chat_media_bubble.dart — _AudioBubblePlayerRegistry is a static singleton holding AudioPlayers; correct, but the waveform rebuilds on every position tick. 

- story_viewer_screen.dart:83 — VideoPlayerController.networkUrl created per story with no preload; hard cut between stories. 

# **Accessibility gaps:** 

- Only in_app_push_banner.dart honors MediaQuery.disableAnimations and provides Semantics. chat_avatar.dart:29 checks disableAnimationsOf for the hero only. 

- No Semantics labels on: send button, + menu, story rings, unread badges, reaction chips, swipe actions. 

- Tap targets: chat_plus_menu label chips, story_viewer reply icons (20px), messages_hub folder chips (36px height) are below 44/48px. 

- chat_money_card amount is FittedBox-scaled — no text-scale accommodation. 

- Contrast: textTertiary at 10–11px on card is used for timestamps throughout. 

# **7. Honest verdict** 

**Against a top-tier messenger (iMessage/WhatsApp/Telegram):** the _ambition_ is there — swipeto-reply, 3-state ticks, reactions, disappearing messages, inline audio with waveform + speed, link previews, optimistic send with nonce dedupe, and a genuinely good money-in-chat card. But the **craft is fragmented** . There are two conversation hubs, two bubble geometries, two composer languages, and four send buttons. A top-tier messenger 

has _one_ bubble, _one_ composer, _one_ list row, tuned to the pixel. Azaman has four of each, each internally decent, none reconciled. The trade chat (chat_interface.dart) is a parallel universe with its own input, its own ticks, and its own hardcoded status colors — it should be the same PremiumMessageBubble + PremiumChatInput with a role flag. 

**Against a premium social app (Instagram/BeReal):** the story _viewer_ is close — segmented progress, tap zones, long-press pause, container-transform open, boosted treatment. But the story _creation_ pipeline is a facade: fake camera preview, text-placeholder video, a no-op highlight viewer, and an analytics screen with hardcoded stat colors. BeReal's restraint (one camera, one flow, one moment) is the opposite of this — Azaman has 8 filters, 4 edit modes, boost, store-link, and highlights, but the core capture is a placeholder. The close-friends and highlights features exist as screens without a working loop. 

# **What would make it class-leading:** 

1. **Collapse to one system.** Delete messages_hub_screen.dart _or_ friends_hub_screen.dart; make chat_interface.dart use PremiumMessageBubble/PremiumChatInput. One bubble radius, one composer height, one row height, one send button. 

2. **Tokenize motion and color.** Extend MotionTokens to every animation; replace all Color(0x…) in chat widgets with AzamanColors roles. Fix the 

two AzamanTheme.light hardcodes immediately — they break dark mode. 

3. **Kill the dead** 

**code** (chat_transfer_sheet, message_status_ticks, reply_preview_bar, thinking_orb, publ ic_profile_modal) and the three unused fields in friend_chat_screen. 

4. **Fix the leaks** : hoist RTCVideoRenderer out of build(); gate the money-card sheen and avatar pulse on visibility (TickerMode/VisibilityDetector). 

5. **Finish the states** : real camera preview, real video preview, working highlight viewer, working call-history dial, error states that aren't Text(e.toString()). 

6. **Accessibility pass** : Semantics on every interactive chat element, honor disableAnimations everywhere (not just the push banner), and lift tap targets to 44px. 

The bones are strong and the money-in-chat card is genuinely distinctive. The gap to classleading is not features — it's **consolidation, restraint, and finishing the loops that are currently stubs.** 

