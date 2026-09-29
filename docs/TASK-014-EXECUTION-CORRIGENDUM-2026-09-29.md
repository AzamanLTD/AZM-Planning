# TASK-014 — EXECUTION CORRIGENDUM (2026-09-29)

**Authority:** This corrigendum is part of the authoritative TASK-014 execution contract for
docs/AZM-buildBrief.md revision f802255f38be7b92643024d7e122c27098d78645.

It supersedes every conflicting TASK-014 instruction below it, specifically the reduced-motion API,
lifecycle placement, AzamanColors getter, ticker-mixin, and fragile grep-count instructions.

## 1. Execution gate

TASK-014 may start only from the current AzamanLTD/AZM-frontend/main after confirming:

- TASK-013 is merged to main (b947a2ba9609cf433c21e2e942d411927ba096e4).
- No other TASK-014 implementation PR/branch is being developed.
- MotionTokens.accessibleDuration(BuildContext, Duration) exists.
- AzamanColors.background exists; backgroundColor does not.
- TASK-006 haptics used by TASK-014 exist.
- The current selector still has no CabinLighting and no deck-slice implementation.
- No TASK-014 changes have already landed on main.

Do not work around a failed dependency. Report the exact failed gate.

## 2. Reduced-motion and lifecycle correction

The original TASK-014 snippets incorrectly call MotionTokens.respectReducedMotion(...).
That method is a static on the compatibility extension ReducedMotion, not on MotionTokens.

More importantly, the reduced-motion lookup reads MediaQuery and must NOT occur from initState.

Use the supported API:

~~~dart
MotionTokens.accessibleDuration(context, normalDuration)
~~~

only from a lifecycle-safe location such as didChangeDependencies() or a build-time path.

### 2.1 Route ribbon

Do NOT construct the controller in initState using MediaQuery.

Use a nullable/late controller initialized once from didChangeDependencies() (or an equivalent
lifecycle-safe pattern), with a guard preventing duplicate initialization. The first initialization
must use:

~~~dart
final duration = MotionTokens.accessibleDuration(
  context,
  MotionTokens.ambient,
);
~~~

Then create the controller with that duration and run the one-shot forward exactly once.

If dependencies change later, update the controller duration without replaying the entrance sweep.

The ribbon's steady state must remain non-repeating.

### 2.2 Bus seat selector deck slice

Apply the same lifecycle rule to _deckSliceController.

Do NOT read MediaQuery from initState.

Initialize/update the slice controller from a lifecycle-safe point using:

~~~dart
MotionTokens.accessibleDuration(
  context,
  const Duration(milliseconds: 420),
)
~~~

The controller count must match the mixin:
- one controller => SingleTickerProviderStateMixin
- two or more => TickerProviderStateMixin

The existing selector already owns other animation controllers, so verify the actual count in the
current file before choosing the mixin. Never guess.

### 2.3 Boarding pass

The boarding-pass state has both the live 1-second ticker and the tear controller.

It therefore MUST use TickerProviderStateMixin, not SingleTickerProviderStateMixin.

Do NOT read reduced-motion from initState.

Initialize the tear duration from MotionTokens.accessibleDuration(context, MotionTokens.standard)
at a lifecycle-safe point. The repeating live-status ticker remains a ticker; it is not a Timer.

## 3. Colour correction

The original _RibbonNode snippet uses nonexistent colors.backgroundColor.

Use the real Azaman palette field:

~~~dart
colors.background
~~~

Do not add a compatibility getter merely to satisfy TASK-014.

## 4. Probe philosophy correction

Several original TASK-014 probes count symbol names rather than production semantics and are
fragile in the presence of comments.

Do NOT treat any exact grep count as an implementation invariant unless the command counts actual
production constructs unambiguously.

For zero-match checks:
- exclude comments where practical;
- otherwise use a targeted source inspection/AST-equivalent check;
- never add/remove prose solely to manipulate a grep count.

The following are semantic invariants, not magic string counts:
- no Timer, Timer.periodic, or Future.delayed in the new hold-ring, demo gateway, route ribbon,
  or boarding-pass implementation;
- no AnimatedSwitcher around the shared InteractiveViewer;
- no raw HapticFeedback remains in the selector;
- no use of colors.backgroundColor;
- no use of String.hashCode for deterministic visuals;
- no second socket/provider/service authority is introduced.

## 5. Route-ribbon motion

The route-ribbon dash sweep is a one-shot entrance animation.

Do not use a repeating animation, periodic timer, or animation started again on every build.

The painter and node positioning must use the same deterministic node geometry.

Sold-out trips remain non-interactive/desaturated.

Loading/error/empty states in transit_trip_list_screen.dart remain unchanged from the existing screen.

## 6. Hold-ring correctness

TransitHoldRing must remain timer-free.

It uses the repeating 1-second AnimationController ticker and an injectable clock seam.

The once-only rules are strict:
- threshold() fires once when the fraction first reaches <= 0.25;
- onExpired is scheduled once when the fraction first reaches <= 0;
- repeated ticks, rebuilds, and duplicate post-frame opportunities must not produce additional calls.

Null expiry renders nothing.

Do not add a release() method to TransitHoldGateway.

## 7. Selection/hold race handling

When _syncSelection changes selection:

- empty selection clears the active hold;
- non-empty selection requests a new hold;
- an older in-flight hold result MUST NOT replace the active hold for a newer selection.

If the implementation of _placeHold needs a selection generation/token to prevent stale async results
from winning, add that small screen-local guard. Do not allow an older asynchronous hold response to
re-arm a ring for seats the user has already deselected or replaced.

The hold failure path must not fabricate a hold expiry; selection remains user-visible and retryable.

## 8. Booking success / keepsake

The booking success sheet stays generic.

keepsake is an optional Widget? slot and must not introduce a transit dependency into the generic sheet.

The transit boarding pass barcode is decorative only. It is not a QR, not a bearer credential,
and must not be presented as scannable.

Use a stable codeUnits-derived seed. Never String.hashCode.

The boarding pass state uses the real TransitBoardingStatus.fromPass semantics already provided
by the existing transit model. Do not invent a new booking state machine in the widget.

## 9. Deck slice

The production backend currently exposes single-deck layouts. Do NOT invent a multi-deck backend
model or change the existing geometry solver.

The deck transition must be implemented so that it works with a temporary local multi-deck
VehicleLayout verification fixture, while normal single-deck production behaviour remains unchanged.

The InteractiveViewer and its single TransformationController remain one live instance.

Do NOT wrap it in AnimatedSwitcher.

The minimap stays outside the slice animation.

## 10. Scope lock

TASK-014 is limited to the files explicitly owned by the task:

NEW:
- lib/marketplace/experiences/transit/demo_transit_hold_gateway.dart
- lib/widgets/seat_selector/transit_hold_ring.dart
- lib/widgets/marketplace/transit_route_ribbon.dart
- lib/widgets/marketplace/transit_boarding_pass.dart
- test/marketplace/experiences/transit/transit_hold_ring_test.dart

MODIFY:
- lib/screens/marketplace/transit_seat_selection_screen.dart
- lib/widgets/seat_selector/seat_canvas_painter.dart
- lib/widgets/seat_selector/bus_seat_selector.dart
- lib/widgets/marketplace/transit_seat_preview.dart
- lib/widgets/marketplace/booking_success_sheet.dart
- lib/screens/marketplace/transit_trip_list_screen.dart

Do NOT touch:
- router
- providers
- services
- pubspec.yaml
- transit contract/model files
- seat geometry solver
- realtime/socket infrastructure
- backend
- unrelated verticals
- TASK-015 or later tasks

## 11. Required regressions

The permanent transit test must prove:
1. fraction math at start/middle/expiry/null expiry;
2. countdown labels at exact second boundaries;
3. repeating-ticker operation with injected clock;
4. threshold haptic exactly once;
5. expiry callback exactly once;
6. null expiry renders no ring;
7. demo gateway completes without Timer/Future.delayed;
8. booking to experience bridge preserves arrival when present;
9. booking to experience bridge supplies the documented 3-hour fallback when arrival is absent;
10. driver to operator mapping is preserved.

Add/extend widget coverage for:
- route node sold-out inertness;
- one-shot route sweep under reduced motion;
- deck-slice transition without a second InteractiveViewer;
- boarding-pass tear commit/reversal;
- deterministic barcode seed;
- CabinLighting null path preserving existing painter behavior.

Where the existing architecture makes direct visual pixel equality impractical, assert stable structural
and behavioural outcomes instead of weakening the invariant.

## 12. Verification

Run:

~~~bash
flutter pub get
flutter analyze
flutter test
flutter build web --debug
~~~

Then run focused TASK-014 tests and inspect every changed file.

For CI, require the exact PR head to pass the repository's authoritative Flutter Quality workflow.

Before requesting merge, report:
- exact head SHA
- changed files
- focused test result
- full test result
- analyzer result
- supported build result
- all semantic probes
- any local environmental crashes (separately from assertion failures)
- confirmation that no open duplicate TASK-014 PR/branch exists

Do not merge TASK-014 as part of implementation work. The senior review will compare the exact head
against current main and independently verify the implementation.

## 13. After implementation

Do not declare TASK-014 complete solely because CI is green.

The next review will independently audit:
- route/list lifecycle and one-shot motion
- hold expiry races and stale async hold results
- selector controller ownership/mixins
- painter null/default behaviour
- booking success keepsake integration
- all public APIs and existing consumers
- actual scope against current main
- demo-mode truthfulness
- deterministic visuals
- accessibility/semantics
- any newly discovered cross-system dependency

Only after that audit should the PR be merged.
