# I1 — Marketplace follows onboarding (5 picks) — design proposal
_Status: PROPOSAL awaiting owner decisions. No implementation started._

Workstream I1 is the only unstarted frontend workstream (I2/I3 are backend and
depend on a contract that does not exist yet). This proposal turns the blocking
"design/naming decisions" into five concrete picks so implementation can start
the moment they land.

## What exists already (verified 2026-10-25)

- Real follow surface, wired and shipped:
  `MarketplaceExtensionsService` (GET `/follows/check/:id`, POST `/follows`,
  DELETE `/follows/:id`, GET `/follows/following?limit&offset`) — used today by
  `business_result_card.dart` and `business_profile_screen.dart`.
- The H rails (`PR #198`) render from the REAL server follow set via
  `followingListProvider`; "From your network" is ABSENT when the follow set is
  empty. So the onboarding's outcome (user follows ≥1 business) is already
  consumable downstream with zero new plumbing.
- Discovery providers (mix offers, nearby search) can seed the pick list with
  REAL businesses — no fabricated suggestions needed.

## Design (recommendations in bold)

**Gate — honest by default, no clocks:**
The onboarding renders ONLY when (a) the real follow set is confirmed EMPTY
(`followingListProvider` = loaded + empty list, NOT loading/error) and (b) the
real discovery feed has at least one row. If either fails, nothing renders.
Demo mode uses the existing marketplace fixtures under the identical gate.

**D1 — Entry point.**
- **A (recommended): dismissible spotlight card** on the Marketplace home,
  below the dial, above the rails. Non-blocking, AURUM restraint.
- B: full-screen one-time overlay at first Marketplace entry (heavier, hijacks
  first-run).
- C: host the CTA inside the network rail's would-be-empty slot instead of a
  separate card.
- Supporting option in all cases: the H rail's empty state can carry a persistent
  "Follow businesses" chip so the affordance never fully disappears.

**D2 — Follow commit.**
- **Per-pick immediate (recommended):** each pick fires the existing POST
  /follows immediately (optimistic toggle with rollback on failure) — same
  grammar as the card's follow button, honest state at every moment, backend
  idempotency already proven.
- Commit-on-done: local selection, batched on Done (adds a two-state UI and a
  failure-with-partial-picks path for no real benefit).

**D3 — Re-prompt policy after "Maybe later".**
- **Never (recommended):** one local dismissal flag, honored forever. The rail
  empty-state chip (D1-C) remains the evergreen recovery route.
- Window variant: re-prompt only if the follow set is still empty after N
  marketplace sessions (needs a session counter; honest but nagging risk).

**D4 — Naming.**
- Sheet title: **"Follow businesses you like"** (alt: "Build your feed").
- Spotlight card copy must stay honest about WHAT following does:
  "Follow a few businesses to light up your From-your-network rail." No social
  proof wording ("friends also follow") — that is I2 territory and stays absent
  until the backend contract exists.

**D5 — Pick list source + demo roster.**
- **Real discovery rows (mix + nearby), ranked by distance then relevance
  (recommended); reuse the existing demo fixture businesses for demo mode**
  rather than a new dedicated roster (fixture consistency with H rails).

**Flow:** spotlight card → bottom sheet (Payout-account sheet grammar: rows,
live check state, Done enabled from 1 pick, "2 of 5" counter as a target not a
gate) → per-pick follows → Done closes → network rail appears with real rows.
"Maybe later" dismisses per D3.

## Test plan (on approval)

- Gate: renders only on loaded-empty follow set + non-empty discovery; nothing
  during loading/error; demo fixtures under the same gate.
- Flow: per-pick mutation via a service seam, rollback on failure, counter
  honesty, Done/later semantics, dismissal flag persistence.
- Regression: H network rail appears once a follow lands; capture honesty
  (prepared vs empty captures, regenerated on the runner only).
- Analyzer parity with main; full marketplace regression battery.

## Hard limits carried forward

- No social-proof wording or friend counts anywhere (blocked on I2).
- No new backend endpoints; only the existing /follows surface.
- No preselection — picks are deliberate, matching the financial-gate
  philosophy even though this is not a money action.
