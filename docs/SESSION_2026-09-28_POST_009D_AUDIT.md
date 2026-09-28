# SESSION 2026-09-28 — Post-009d Balance-Card Audit & Resolution

## Scope

Independent review of the TASK-009d balance-card rebuild (PR #103, merge
`ee60455`) before lifting the hold on TASK-010. The audit covered the three
merged widget files (`hologram_balance_card.dart`, `flippable_balance_card.dart`,
`odometer_number.dart`), the front face's delta chip, the flip gesture, the
back-face breakdown grid, and the susu derivation semantics.

## Findings and dispositions

| ID | Finding | Priority | Disposition |
|----|---------|----------|-------------|
| D-1 | Delta chip rendered change magnitude (`+USDC 50.00`) **while the balance was masked**, defeating the eye-toggle privacy affordance in public. | P1 | **Fixed — PR #104** |
| D-2 | Flip used a fixed 450 ms spatial transition and the chip a fixed slide; reduced motion (brief acceptance #11) was not honoured. | P2 | **Fixed — PR #104** |
| D-3 | While `susuListProvider` had no data (loading **or** failed), the Susu bucket rendered a greyed `0.00` — "unavailable" read as "nothing committed". | P2 | **Fixed — PR #104** |
| D-4 | Back-face amounts were inflexible and `AzMoney.amount` never compacts; a large balance at large text scale could overflow the fixed cell. | P2 | **Fixed — PR #104** |
| — | Susu socket-refresh staleness (list can go stale without a socket nudge). | P3 | Deferred (tracked in buildBrief). |
| — | Flip gesture has no `Semantics` action (pre-existing, not introduced by 009d). | P3 | Deferred. |
| — | F-018 dead files still present. | — | Separate decision, not part of 009d. |

## Resolution — PR #104 (`93ed294`, squash of `fix/balance-card-audit-followups`)

- **D-1**: the chip renders only when the balance is visible
  (`chipDelta = isVisible ? _delta : null`). Tracking continues while hidden so
  the baseline stays honest; only rendering is suppressed.
- **D-2**: the flip re-reads `MediaQuery.disableAnimationsOf` on every toggle
  and collapses the controller duration to zero under reduced motion; the chip
  animations use `MotionTokens.accessibleDuration` (the established
  convention).
- **D-3**: `_BackFace` now receives `susuKnown` (`susuAsync.hasValue`); an
  unknown Susu bucket renders an em dash, never a zero.
- **D-4**: back-face amounts are `Flexible` + `FittedBox(scaleDown)`, mirroring
  `OdometerNumber`'s guard. Values are unchanged; only scale down when needed.

## Regression coverage — `test/widgets/balance_cards_test.dart` (6 tests)

1. Chip appears on a balance change while visible (positive control).
2. Chip does **not** appear on a change while hidden (the D-1 privacy pin).
3. Flip lands on the first frame under reduced motion (one `pump()`, no
   settle).
4. Unloaded susu list renders a dash, never a zero.
5. Only an ACTIVE member of an ACTIVE group counts toward the committed total
   (pendingContract and completed-group memberships excluded).
6. A 1,234,567.89 balance in a 320x180 card does not overflow (no
   `RenderFlex` exception).

## CI evidence

- Branch `fix/balance-card-audit-followups` @ `2982ef0`: **Flutter Quality**
  success — 0 analyze errors, 0 lints in touched files, **609/609 tests**
  (603 prior + 6 new).
- Main @ `93ed294`: **Android Integration** success (full battery re-run per
  the audit gate).

## State after this session

- 009a–009d are DONE and green on main. The TASK-010 hold from the audit is
  lifted: **GO for TASK-010 (animated nav transition)**.
- buildBrief mission table and F-020..F-023 defect rows updated to RESOLVED.
- Deferred P3s (susu socket staleness, flip Semantics) remain tracked in the
  buildBrief for the next pass.
