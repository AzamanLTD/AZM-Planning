# AZAMAN MASTER PRODUCT REDESIGN — MASTER HANDOFF
_Last refreshed: 2026-10-25 ~15:45 UTC (second refresh — post-head-updates)_

## Repository state at last refresh

- **AZM-frontend main = `190f722`** (`feat(demo): labelled BalanceCardPair preview on demo Home` — PR #190's content, owner-merged). Post-merge CI green.
- **AZM-backend main** — not re-verified this session (no backend change in this session's scope).
- Workstream H was re-implemented from scratch this session after the previous agent's uncommitted H work was lost with its sandbox.

## Workstream status

| Workstream | PR | Branch / head | Status |
|---|---|---|---|
| A — Home persistent translucent header + pull-down notification centre | #191 | `feat/home-persistent-header` **`8326701`** | PR open, head updated 2026-10-25 PM (grouped-list fix, see below); android green on `8326701`; quality awaits #199 merge (golden drift, resolved by #199) |
| B — Shared launcher surface: Send rise + X + pull-dismiss | #192 | `feat/send-surface-grammar` `33c78e4` | PR open, both CI lanes green |
| C — Withdraw Add-Cash grammar + swipeable payout sections | #193 | `feat/withdraw-surface-grammar` `51d42d3` → **`de2a8b1`** | PR open; head updated 2026-10-25 with the **owner wallet-selector correction** (see below); both lanes green on `de2a8b1` (quality 121154042864, android 121154042823) |
| D — Binance-style Convert presentation, execution gated | #194 | `feat/convert-presentation` **`146166e`** | PR open, head updated 2026-10-25 PM (Image A review + honest Max fill, see below); android green on `146166e`; quality awaits #199 merge (golden drift, resolved by #199) |
| E — Balance card E1 grammar + E2 320dp overflow + E3 back-face rhythm + E4 confetti | #195 | `feat/balance-card-polish` `4991ea5` | PR open, both CI lanes green |
| F — Story rail in-place expansion + viewer caption reveal | #196 | `feat/marketplace-story-expansion` `13cfac0` | PR open, both CI lanes green |
| G — Business list declutter + avatar badge/bookmark/story ring | #197 | `feat/business-list-identity` `c73bd7b` | PR open, both CI lanes green |
| H — Mix co-activates Near You, local-first rails | **#198** | `feat/mix-local-first` `6351bc5` | **NEW this session** — PR open, both lanes green, mergeable clean |
| I1 — Marketplace follows onboarding (5 picks) | — | not started | blocked on design/naming decisions, data honest by default |
| I2 — Backend social-proof contract (friends/friends-of-friends counts) | — | not started | **missing contract** — no backend work done |
| I3 — Frontend integration with social-proof contract | — | not started | depends on I2 |
| — old balance-card draft #185 | #185 | `feat/balance-card-fx-status` `534a896` | stale draft, superseded by #190/#195 — do not resurrect |

## Owner correction delivered (2026-10-25)

**Withdraw popup wallet selector (PR #193, head `de2a8b1`)**: the MoMo section no longer stacks both saved accounts (the "two networks" demo fixture: MTN *My MTN* + Telecel *Telecel Acct*). It now renders ONE Add-Cash-style selector: a `Choose payout account` row until a deliberate pick, then a compact selected-account row (network badge, verified name, `024 ••• 456` phone, chevron) that opens the `Payout account` bottom sheet with per-account rows, live check state, and the add-account CTA. No preselection (deliberate-pick financial gate preserved), no wire-contract change, locks while submitting. Evidence: comment 6397080599 on #193. All six prepared captures regenerated; all six empty captures byte-identical.

## Workstream H detail (PR #198)

- `resolveSilently()` no-prompt seam on `NearbyLocationResolver` (services+permission already granted → position; otherwise throws WITHOUT prompting; `resolve()` keeps its in-context ask).
- Pressing Mix → silent resolve → co-activates Near You (pill visibly active, proximity strip "Nearby results on", nearby feed fires with current query/category) when location is already available; silent failure → honest off state, no prompt, no loop. The Near You pill remains the explicit request/retry route and recovers normally.
- Local-first rails, real-data-only: "Nearby for you" (nearbySearchProvider rows, real distanceKm, km formatting matches business cards) and "From your network" (mix offers ∩ the REAL server follow set via `followingListProvider`; empty → rail absent). Coordinates are never rendered.
- Dial category inside Mix+NearYou re-fires the nearby search; pull-to-refresh refreshes BOTH feeds; exiting Mix restores the pre-Mix dial wire and turns off only the mix-owned proximity feed (explicitly-enabled Near You survives).
- Tests: new `marketplace_mix_local_first_test.dart` 7/7; full regression battery green (see comment 6397181743).

## Known defects / findings (pre-existing, out of scope, documented)

- Stories-rail empty state overflows its fixed 94px rail extent by 18px (clipped, invisible; only surfaces with deterministic fixtures). Surfaces under #196's scope.
- `use_build_context_synchronously` info in withdrawal_screen.dart durable-recovery dialog (pre-existing on #193's original head; unchanged).
- Capture generator (`tool/captures/*`) deadlocks at teardown in the sandbox after writing its PNG — documented in the generator header; CI's battery never picks tool/ files.

## Missing contracts / decisions

- **I2 social-proof backend contract** — nothing exists; no "friends visited/followed" wording anywhere until it does.
- Convert execution remains gated (`kConvertExecutionSupported = false`) pending the server-authoritative quote/accept contract.
- **RESOLVED**: the Binance Convert reference (IMG_0345) arrived 2026-10-25 12:07 UTC and was reviewed in full.

## Head updates delivered 2026-10-25 PM

**PR #191 → head `8326701`** (branch fast-forwarded, stray `pr191` ref deleted):
- Fixed a REAL grouped-list defect: the panel built Today/Earlier sections from
  `groups.length` positions of `sectionOrder`, so any notification older than today
  silently vanished from the panel while the badge still showed the total
  (aggregation math was right, presentation lied).
- Honest unread counting: badge now derives from the same aggregation as the list
  (Today + Earlier unread), never a raw `notifications.length`.
- Cards are tappable via the shared dispatcher (same route as `NotificationCard`),
  with semantic labels; swipe action relabelled **Hide** (honest delete: local
  dismissal, no backend endpoint claimed).
- 41/41 tests locally; evidence comment 6398834335 on the PR.
- CI: android GREEN on `8326701` (run 40201975406). Quality was CANCELLED mid-test
  (see anomaly below); rerun also cancelled.

**PR #194 → head `146166e`** (on top of `f4c7a13`):
- Image A (IMG_0345, Binance Buy/Sell) reviewed in full: From card with balance
  + Max, overlapping center swap toggle, To card, full-width CTA.
- Closed the one fidelity gap: **honest Max quick-fill** on the From panel — renders
  only in forward direction when the canonical USDC read has actually loaded and is
  a sane positive figure; fills the same figure the balance line shows at display
  precision; zero balance → no Max; pure text edit, execution gate stays closed.
  Reversed direction omits it (send-screen principle: never silently re-denominate).
- 6 new regression tests (48/48 across the convert battery locally); the 6 honest-default
  goldens pass UNCHANGED (unresolved read → no Max button in captures — no regen needed).
- CI on `146166e`: both lanes cancelled mid-run (see anomaly).

## CI cancellation anomaly (2026-10-25 ~15:22-15:37 UTC, needs owner attention)

Three runs across two PRs and both workflows were CANCELLED mid-execution within
~15 minutes of each other, with no code cause (all steps before the cancel were green):
- PR194 quality run 40205389812 — cancelled at 15:22:07 mid "Test with coverage".
- PR191 quality run 40201975498 attempt 1 — cancelled at 14:38:25 mid test; my rerun
  (attempt 2) also cancelled at 15:24:11, ~5 min in.
- PR194 android run 40205389748 — cancelled at 15:37:07 mid "Build Android debug APK",
  30 min in.
No concurrency conflict exists (groups are per-workflow-per-PR-ref; no superseding runs
in either group). This pattern matched an org Actions-minutes/billing wall (in-flight
jobs get cancelled) — **the owner resolved the underlying cause in the afternoon**
("I know what happened, I have worked on it now").

**RESOLVED 2026-10-25 ~16:50 UTC.** Once runs could complete again, both quality
lanes failed on a SECOND, distinct issue: the ubuntu runner image rolled its glyph
rasterisation between 11:37 and 16:02 UTC — the native_amount `gallery_*` goldens
(untouched by either PR; green at 11:37 in run 40189169844) failed at 0.03-0.05% /
85-86px in the typed "20" text. Proof it was environmental: an `update_goldens`
dispatch on main `190f722` (run 40210222370) regenerated every non-gallery golden
**byte-identical**; only the 8 gallery captures changed. Those 8 regenerated goldens
went up as **PR #199 (head `5f2afc4`, branch `chore/native-amount-gallery-golden-rebaseline`)
— BOTH LANES GREEN** (full suite incl. all 40 native_amount captures). Once the owner
merges #199, #191 and #194 re-fire against the new merge ref and should go fully green
(no overlap, no rebase needed); their android lanes are already green on `8326701`
and `146166e` respectively. Evidence comments: 6400108278 (#199), 6400108402 (#191),
6400108542 (#194).

## Confirmations

- **No PR merged. No deploy. No production/backend change. Main untouched.** All work is on feature branches with open PRs awaiting owner review. CI verified on the exact head of every PR listed above.
