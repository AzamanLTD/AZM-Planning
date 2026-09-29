# TASK-015 — EXECUTION CORRIGENDUM (2026-09-29)

**Authority:** This corrigendum is part of the authoritative TASK-015 execution contract for
docs/AZM-buildBrief.md revision 9f7f0c19827295c523d91b2dba898d87a9d9b717.

It supersedes every conflicting TASK-015 instruction below it, specifically the invalid
`MotionTokens.respectReducedMotion` calls and the fragile case-sensitive `rg -c` symbol-count
probes whose expectations assume snake_case import lines match CamelCase symbol names.

## 1. Execution gate

TASK-015 may start only from the current AzamanLTD/AZM-frontend/main after confirming:

- TASK-013 is merged to main (b947a2ba9609cf433c21e2e942d411927ba096e4).
- TASK-014 is settled (PR #111 merged or explicitly closed) or the owner directs overlap;
  TASK-015 touches `hotel_booking_screen.dart` only, but both tasks share the dossier sheet,
  tempo enums, and haptic vocabulary, so the reviews should not interleave on one head.
- No other TASK-015 implementation PR/branch is being developed.
- All eight of the task's pre-flight probes pass (each was re-verified against current main on
  2026-09-29 — see §4).

Do not work around a failed dependency. Report the exact failed gate.

## 2. Reduced-motion API correction

The TASK-015 snippets call:

~~~dart
MotionTokens.respectReducedMotion(context, MotionTokens.spatial)
~~~

That method is a static on the compatibility extension `ReducedMotion`, not on `MotionTokens`;
`MotionTokens.respectReducedMotion(...)` does not compile. Use the supported API:

~~~dart
MotionTokens.accessibleDuration(context, MotionTokens.spatial)
~~~

Exactly two call sites need the change, both inside `HotelArrivalSheet`'s
`didChangeDependencies` (the door controller and the key-card controller). Their placement is
already lifecycle-safe: no MediaQuery read occurs in any `initState` in the TASK-015 snippets,
and the reduced-motion lookup in `didChangeDependencies` is legal. Only the symbol changes; the
durations (`spatial` for the door, `emphasized` for the card) stay as written. Also correct the
TEST-SAFETY comment that names `respectReducedMotion` — it should read `accessibleDuration`.

## 3. Probe corrections (snake_case imports never match CamelCase names)

Import lines are snake_case (`import '.../hotel_arrival_sheet.dart';`), so a CamelCase symbol
name never matches the import. Every `rg -c` expectation that counted "import + usage" is
inflated by exactly one. The Step 6 verification block is corrected to:

```bash
# Deletions (unchanged — expect no matches):
rg -n "_RoomExplorer|_RoomTile|_RoomDetailCard|_BookingPanel|_hotelAmenityIcon" lib/screens/marketplace/hotel_booking_screen.dart   # expect: 0
rg -n "BookingSuccessSheet|flutter_animate|ChoiceChip" lib/screens/marketplace/hotel_booking_screen.dart                              # expect: 0

# Mount points — semantic probes, not counts:
rg -n "HotelArrivalSheet.show\(" lib/screens/marketplace/hotel_booking_screen.dart     # expect: 1 (the show call; import is snake_case)
rg -n "BuildingCrossSection\(" lib/screens/marketplace/hotel_booking_screen.dart       # expect: 1 (constructor usage)
rg -n "StayDateRibbon\(" lib/screens/marketplace/hotel_booking_screen.dart             # expect: 1
rg -n "StaySummaryBar\(" lib/screens/marketplace/hotel_booking_screen.dart             # expect: 1
rg -n "RoomDossierContent\(" lib/screens/marketplace/hotel_booking_screen.dart        # expect: 1
rg -n "showMarketplaceDossierSheet\(" lib/screens/marketplace/hotel_booking_screen.dart   # expect: 1
rg -n "MarketplaceDetailPresentation\.roomDossier" lib/screens/marketplace/hotel_booking_screen.dart   # expect: 1
rg -n "MarketplaceMotionTempo\.relaxed" lib/screens/marketplace/hotel_booking_screen.dart              # expect: 1
rg -n "AzamanHaptics\.selection\(\)|AzamanHaptics\.threshold\(\)" lib/screens/marketplace/hotel_booking_screen.dart   # expect: 2 (one each)
rg -n "AzText\.(title|caption|money)\(" lib/screens/marketplace/hotel_booking_screen.dart   # expect: >= 2 (title + caption)
rg -n "AzSpace\.sm" lib/screens/marketplace/hotel_booking_screen.dart                      # expect: >= 1
rg -c "Future.delayed" lib/screens/marketplace/hotel_booking_screen.dart                    # expect: 1 (the preserved 3s push — must stay exactly 1)
```

The Stay-Date-Ribbon Step 3 probe

```bash
rg -c "ribbonNights|ribbonDateAt|stayTotalFor" lib/widgets/marketplace/stay_date_ribbon.dart   # expect: 14
```

is comment-sensitive (doc comments name the helpers). Drop the count; the anchored declaration
probe on the line above it is the authoritative one:

```bash
rg -n "^int ribbonNights|^DateTime ribbonDateAt|^double stayTotalFor" lib/widgets/marketplace/stay_date_ribbon.dart   # expect: 3
```

All other TASK-015 probes (class declarations, `Timer|Future.delayed` zero-expects, the
`Chip(` zero-expect in the dossier content) are correct as written and stay.

## 4. Fresh-main anchor verification (2026-09-29)

Re-verified against AzamanLTD/AZM-frontend at b947a2b (TASK-013 merged):

- `hotel_booking_screen.dart` is now **594** lines (the preamble says 595 — cosmetic drift,
  no anchor depends on the count). `_RoomExplorer` (L301), `_RoomTile` (L405), and
  `_BookingPanel` (L495) exist; `ChoiceChip`, `showDateRangePicker`, and `BookingSuccessSheet`
  are present.
- `abstract final class AzText` / `AzSpace` / `AzRadius` exist.
- `AzamanHaptics.selection/threshold/commit/confirm` exist (TASK-006 landed).
- `showMarketplaceDossierSheet` exists (`marketplace_dossier_sheet.dart` L41).
- `MarketplaceMotionTempo { relaxed, balanced, quick }`, `MarketplaceDetailPresentation.roomDossier`,
  and `MarketplaceNavigationMode.floorTraverse` exist in the blueprint; the hotel preset ships.
- `PremiumGlassContainer` exists (`premium_glass_container.dart` L39).
- `MotionTokens.control/enter/spatial/emphasized` all exist.
- `ThemeProvider.getColors(AzamanTheme.dark)` exists; `AzamanColors` is an instance class — the
  snippets' `final AzamanColors colors;` constructor params are correct usage, no static misuse.
- `HotelRoom` carries `roomNumber`, `floor` (nullable int), and `weekendPriceUsdc` (nullable
  double) exactly as the snippets assume.

## 5. Verification contract

Unchanged from the task body: G3 is the supported A.5 Android target. The repository is
mobile-only (no `web/` directory) — no web build is claimed. Focused new-suite tests plus the
full suite and analyzer under the CI flags must pass before the PR is posted for independent
exact-head review; the PR stays open, unmerged, until that review signs off.
