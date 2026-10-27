# 2026-10-27 — Backend PR consolidation (financial core, marketplace, notifications)

**Scope:** AZM-backend open-PR consolidation and review. No merges — all PRs left open for owner merge decision. `destinationType` and dust-threshold decisions remain **visibly pending**; no financial paths merged or activated.

## Consolidated PRs — review-ready (base `main`, both native CI lanes green at the exact head)

### #369 — Financial core (consolidates #367 + #368)
- **Head:** `7715854` — CI: Test Suite run **1444** success, financial-durability run **359** success (production commit mode).
- Native-GHS wallet + FX rate snapshot + backfill valuation (#367), universal fail-closed USDC transaction-route gating (#368), plus the hardening commit: DB-origin `config.baseCurrency` reaching `settlementScaleFor()` in the payout/contribute legs is now rejected with an explicit `UNSUPPORTED_BASE_CURRENCY` 400 (previously the unvalidated boundary), with two real-PostgreSQL regression tests in the single-winner suite.
- Supersedes: #367, #368 (marked, kept open for reference).

### #377 — Marketplace personalisation (consolidates #370-#373)
- **Head:** `58ea90f` (= `08f427d` + 2-line release-chain repair) — CI: Test Suite run **1447** success **3732/3732** (383 suites), financial-durability run **362** success 584/584.
- Scope: user-scoped business bookmarks (#370), authoritative active/complete story state (#371), real friends/friends-of-friends social proof (#372), persistent "Show less" recommendation feedback (#373). Union merges with conflict resolution in routes, release chain (`infra/autoRelease.js`, boot installer registration) and the offered-categories overlay path.
- **Defect found and fixed during verification:** the conflict union inserted the recommendation-feedback overlay block before the bookmark catch block's closing `}` in `infra/autoRelease.js` -> parse error -> first CI attempt (`08f427d`) failed BOTH lanes (9 suites / 10 tests cascade, plus 1 suite in the financial lane). Repaired at `58ea90f`; all 10 CI-failing suites re-verified locally on real PostgreSQL 15 (serial, CI semantics) before push.
- Supersedes: #370, #371, #372, #373 (marked, kept open for reference).

### #378 — Notifications (consolidates #374 + #376)
- **Head:** `b6dde9d` — CI: Test Suite run **1446** success **3711/3711**, financial-durability run **361** success 584/584.
- Scope: owner-scoped soft notification dismissal (#374) — record preserved (audit trails intact), hidden from list in all modes and from unread count, owner-scoped mutation (foreign id -> same honest 404 as unknown id), idempotent flag preserving the FIRST `dismissedAt`, converges with concurrent markAsRead, socket `notifications_updated` for multi-device badge sync, fail-closed boot overlay; and the payload/category contract repair (#376) — consistent 404 contract, repaired `actionPayload` shapes/categories at every producer, `STOREFRONT` added to `_normalizeCategory` (no longer silently downgraded to GENERAL).
- Zero merge conflicts (#374/#376 touched disjoint regions of `notificationService.js`).
- **Delivery honesty:** dismissal is a soft, synchronous flag operation. Delivery to Firebase/socket remains best-effort — no outbox, retry queue, or guaranteed-delivery claim was introduced.
- Supersedes: #374, #376 (marked, kept open for reference).

## Verified standalone

### #375 — Read-only deposit continuation (session-interruption recovery)
- **Head:** `d76beac` — CI: Test Suite run **1437** success, financial-durability run **350** success. Own PR, untouched by consolidation.

## Untouched, outside this scope
- #360 (trade-account rails wiring), #319 (architecture note).

## Merge order note

#369, #377, #378 are each base-`main` + their own scope; they are independent of each other, so any merge order works. #375 rides independently. Squash-merge convention applies (see ledger history). After any merge, re-run post-merge main-head CI per the standing completion policy before recording it as verified in the ledger.

## Frontend dependency note

AZM-frontend PR #203 (joystick + balance card, head `cf28c15`, CI green) depends on #367 (native-GHS valuation), #370 (bookmarks), #371 (story state), #373 (show less) — all now carried by #369 and #377. #203 stays unmerged until its backend dependencies land.

## Process lessons (recorded for future consolidation waves)

1. Conflict unions on `infra/autoRelease.js` must be followed by `node --check` before push — a single misplaced brace took down both CI lanes through the boot-import cascade.
2. Never run a full-suite jest in the background while verifying a focused suite against the same disposable DB — the sweeper deleted test users mid-suite and produced phantom 401s (`USER_GONE`) that looked like real auth defects.
3. After switching branches locally, regenerate the Prisma client from that branch's schema before running suites — a stale client from another branch's schema fails on missing columns and mimics real regressions.
