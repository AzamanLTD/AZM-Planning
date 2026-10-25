# 2026-10-25 Backend contract audit & composability — PRs #367–#376 (all separate, none merged)

Audit performed against live `AzamanLTD/AZM-backend` `main` @ `0edefa1` and each PR's live head on 2026-10-25 (18:50–19:30 UTC). No PR was merged, auto-merged, deployed or written to `main`. One focused follow-up PR (#376) was opened to fix real contract breaks found by the audit.

## 1. Live PR state (heads re-verified via GitHub API, not the historical snapshot)

| PR | Head | Base | CI at head |
| --- | --- | --- | --- |
| #367 native-GHS wallet/FX | `9e83a909` | `0edefa1` | Test Suite ✅ + financial-durability ✅ |
| #368 USDC fail-closed gate | `2c826fa2` | `9e83a909` (stacked on #367) | ✅ + ✅ |
| #369 susu payout/cancel | `f44b00e1` | `2c826fa2` (stacked on #368) | ✅ + ✅ |
| #370 business bookmarks | `46d7ed22` | `0edefa1` | ✅ + ✅ |
| #371 active/unseen stories | `072409ec` | `0edefa1` | ✅ + ✅ |
| #372 social proof | `328387f7` | `0edefa1` | ✅ + ✅ |
| #373 show-less ranking | `bf69eb58` | `0edefa1` | ✅ + ✅ |
| #374 notification dismissal | `d1945c93` | `0edefa1` | ✅ + ✅ |
| #375 deposit continuation | `50c902ca` | `0edefa1` | ✅ + ✅ (matches the historical snapshot) |
| #376 (NEW) notification payload contract | `c2d4798` | `0edefa1` | see §2 |

Dependency structure: **#367 → #368 → #369 is an intentionally stacked financial chain** (each base is the previous head). #370–#375 are all independently based on `main` @ `0edefa1`. The five marketplace PRs share additive edits in `package.json` (release notes), `.github/workflows/test.yml` (installer steps), `DEPLOY_RUNBOOK.md`, `infra/autoRelease.js`, `src/routes/index.js`, `prisma/schema.prisma` — every later merge must re-union its own additive entries (order documented in §5).

## 2. PR #376 — the audit's one real backend gap, fixed (Priority E)

`docs/notification-payload-contract.md` (in the PR) is now the single source of truth for the notification ecosystem: 13 `NotificationCategory` enum values with semantics, a 40-row canonical action → payload matrix, ownership/privacy rules, and legacy/malformed/stale payload behaviour. No second taxonomy was introduced; client dispatch remains `action`-keyed, never raw `route` strings.

Four producers were provably broken (writes to nonexistent `type`/`metadata` columns, `message:` where `body:` is required, non-enum categories, `STOREFRONT` missing from the normalize allow-list). All four are fixed with canonical payloads; a 22-case contract suite pins emitters against the documented matrix, and a static-invariant test blocks any new producer from re-introducing invalid writes.

One durability-lane test (`payout-unknown-outcome.test.js` case D) pinned the old broken call shape (top-level `type` on a call using `message:` — a payload that could never persist); commit `c2d4798` re-pins it to the canonical `WITHDRAWAL_STATUS` payload that actually delivers. Local verification (disposable PostgreSQL 15): payout-unknown-outcome + contract suite 26/26, adjacent withdrawal-reconciliation/r15c/r15d/r15h suites 28/28. CI at `c2d4798` is fully green (runs 40224653715 / 40224653949): **Azaman Test Suite 3700/3700 tests passed**, route-registry verification against the frontend PASS, database backup/restore recovery drill SUCCESS, `npm audit` 0 vulnerabilities, and the financial-durability lane green. Earlier heads failed only on the audit step fixed by the same `fix(deps)` advisory patch (body-parser/multer/zod) that #370–#375 each carry.

## 3. Notification dismissal (#374) re-confirmed (Priority C)

All seven contract points are test-pinned in `__tests__/notification-dismissal.test.js` and re-ran green on the composed tree: owner-only dismissal (403 otherwise, no existence leak — 404 for unknown id), soft + idempotent (record preserved, payload untouched), `isRead`/`isDismissed` independence with explicit flags, hidden from list AND unread-count, `DISMISSED` socket event on the owner's user room (multi-device convergence), read/dismiss concurrency converges in either order, and no mutation of the underlying transaction/audit records. No fake server notifications were introduced for local drafts.

## 4. Composability of #370–#375 (+ #376) — proven on a disposable DB (Priority D)

- **Sequential merge:** #370 → #371 → #372 → #373 → #374 → #375 → #376 merges cleanly with six documented union resolutions (release-note line, `test.yml` installer steps, `DEPLOY_RUNBOOK.md`, `autoRelease.js`, `routes/index.js`, `schema.prisma`). `autoRelease.js` and `routes/index.js` needed manual structural repair during composition (closing-brace/mount-order), reflecting what a real rebase will hit.
- **Installer chain from an empty DB:** all **22 installers** (the 18 established overlays + the new ones from #370/#373/#374) passed on an empty disposable PostgreSQL 15 database, and a **repeat run** also passed — additive overlays are idempotent per repo convention. All new artifacts verified present: `BusinessBookmark`, `BusinessRecommendationFeedback`, `Notification.isDismissed`/`dismissedAt` + `Notification_userId_isDismissed_idx`.
- **All PR test suites together on the composed tree:** #370–#373 = **44/44**; #374+#375 = **22/22**; with #376's contract suite = **31/31**; adjacent withdrawal-reconciliation suites = 28/28.
- **Contract spot-checks:** #370 keys on canonical `BusinessProfile.id` with authenticated-owner scope; #371 reuses `Story`/`StoryView` authority (two batched queries per page, read-only, privacy-safe booleans); #372 counts distinct eligible friends/FoF who follow the business (no duplicate paths, no viewer self-count, no "visited" claim); #373 is per-user, idempotent, reversible, and feeds the ranking query.

## 5. Required merge/update order (documented so no later PR loses an installer/route/release step)

1. Merge #367, then rebase #368, then rebase #369 (stacked chain — #368's base is #367's head).
2. #370–#375 are order-independent among themselves, but each merge changes the same six shared files: the next PR in the queue must rebase and re-union **its own** additive entries — release-notes line in `package.json`, its installer line in `test.yml` **and** `infra/autoRelease.js`, its route mounts in `src/routes/index.js`, its schema models, its runbook section. (The composed tree proves the union is what CI expects.)
3. #376 similarly re-unions `docs/` + the two notification test files; it touches `notificationService.js` in regions disjoint from #374, so they merge cleanly either order.
4. Runbook: never run overlays against production; provision from empty via the full installer chain (which CI itself rehearses).

## 6. Operation recovery audit (Priority F) — no new endpoint beyond #375

| Operation | Authoritative re-observation route | Durable identifier | Permitted states |
| --- | --- | --- | --- |
| Fiat deposit (Moolre MoMo) | `GET /api/deposit/fiat/status/:transactionId` (**#375**, read-only, owner-scoped, 404 non-leak) | `TransactionHistory.id` (`DEPOSIT_FIAT` row) | `PENDING` / `COMPLETED` (`settledAt` set) / `FAILED` / `FROZEN_DISPUTE` |
| Crypto deposit | provider webhooks → `GET /api/finance/transactions?type=DEPOSIT_CRYPTO` | `TransactionHistory.id` | same ledger enum |
| Withdrawal (fiat/crypto) | `GET /api/withdraw/status/:reference`, `GET /api/wallet/withdraw/status/:withdrawalId`, history lists | `Withdrawal.id` / provider reference | PENDING → lifecycle state machine |
| Send / internal transfer | synchronous idempotent POST (`failurePolicy RELEASE`); replay same `Idempotency-Key` returns the committed body — never a second transaction; verify via transaction history + receipt route | `TransactionHistory.id` + `FinancialOperation` row (permanent, TTL-prunable response cache only) | COMPLETED rows |
| P2P trade / escrow | `GET /api/trades/:id`, `GET /api/trades/active`, `GET /api/trades/history`, `GET /api/escrow/ticket/:ticketId`, `GET /api/p2p/action-required` | `Trade.id`, escrow ticket id | trade state machine |

Gap conclusion: only the fiat-deposit continuation was missing (no ownership-checked way to re-observe one in-flight deposit with its payment-instruction metadata) — exactly what #375 adds, read-only. **No generic cancellation endpoint, no ledger entries, no server-side draft state, and no new operation types were created.** A local form draft stays client-local; an already-submitted pending payment never loses server responsibility (ledger row persists; idempotency prevents doubles).

## 7. Frontend contract handoff (exact heads; all PRs unmerged)

- **#370** `POST/GET /api/bookmarks`, `GET /api/bookmarks/check/:businessProfileId`, `DELETE /api/bookmarks/:businessProfileId` — Bearer; keys are canonical `BusinessProfile.id` (uuid); response `{ bookmarks: [...] }` / `{ bookmarked: bool }`; idempotent toggle on POST.
- **#371** — no new endpoints: `GET /api/business/search` (optionalAuth) and `GET /api/follows/following` (protect) each return a per-business `storyState: { hasActiveStory, hasUnseenStory (null when anonymous), latestStoryAt }` block; booleans only, no story contents.
- **#372** `GET /api/social-proof/:businessProfileId`, `POST /api/social-proof` (batch `{ businessProfileIds: [...] }`) — Bearer; counts distinct direct friends and friends-of-friends who follow the business; never counts the viewer; no "visited" claim.
- **#373** `POST /api/recommendation-feedback` (show-less), `GET /api/recommendation-feedback`, `GET .../check/:businessProfileId`, `DELETE .../:businessProfileId` — Bearer; per-user, idempotent, reversible via DELETE; affects subsequent recommendation ranking only — never following/bookmarks/reviews.
- **#374** `DELETE /api/notifications/:id` — Bearer; soft dismissal, idempotent, 404 unknown (no existence leak), 403 foreign row; dismissed hidden from `GET /api/notifications` and `GET /api/notifications/unread-count`; socket `DISMISSED` on the owner room; `isRead` untouched/independent.
- **#375** `GET /api/deposit/fiat/status/:transactionId` — Bearer; read-only; `200 { success, transaction: { id, type, status, amountUsdc, createdAt, settledAt }, continuation: { provider, network, amountGhs, amountGhsExact, quoteId, quoteExpiresAt, rateAtInitiation, payerPhoneMasked, providerRef } }`; `404 { success: false }` for unknown/foreign/non-DEPOSIT_FIAT; raw MSISDNs never returned. Frontend distinguishes: local unsubmitted draft (client-only) vs submitted-pending (200 PENDING) vs ambiguous submission (re-observe this route / replay the idempotent POST — the ledger row is the truth).
- **#376** — no new client surface; fixes four producers so the payloads the frontend already parses actually persist. Full matrix: `docs/notification-payload-contract.md` in the PR.

Statuses/errors, null semantics and payload examples for each are in the PRs' own contract docs and test files; the matrix in #376's doc is the canonical cross-reference. All PRs remain open and independently reviewable.
