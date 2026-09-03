# AZAMAN Production Release Gate

**No release sign-off from vibes. Every checked item needs evidence.**

## A. Repository integrity

- [ ] All five `main` SHAs recorded.
- [ ] No unresolved duplicate implementation PRs.
- [ ] Stale branches reviewed and deleted when safe.
- [ ] All changed repos have exact-head CI green.
- [ ] Final diffs audited after CI.

## B. Financial correctness

- [ ] Wallet/escrow/trade/withdrawal/order/invoice/reservation/dine-in mutations audited.
- [ ] Transactions/conditional writes/idempotency verified.
- [ ] Concurrent duplicate attempts tested.
- [ ] Ledger/balance/history effects reconciled.
- [ ] Provider ambiguity/retry recovery tested.
- [ ] Admin force financial actions tested.

## C. Security

- [ ] Authentication/session refresh/revocation verified.
- [ ] Tenant boundaries tested for positive and cross-tenant cases.
- [ ] Actor/role/permission matrix verified.
- [ ] Suspended/inactive staff access revoked as designed.
- [ ] IDOR/replay/privilege-escalation red-team cases passed.
- [ ] No secrets in source/logs/events/audit data.

## D. State + contracts

- [ ] State machines documented and enforced.
- [ ] Generic CRUD cannot bypass side-effectful transitions.
- [ ] Cross-repo API contracts traced and executable where practical.
- [ ] Dine-in contract verified end-to-end.
- [ ] Experience Blueprint contract verified.
- [ ] Notification/realtime payloads and consumers verified.

## E. Realtime/recovery

- [ ] Events emitted only after authoritative commit.
- [ ] Duplicate events are safe.
- [ ] Reconnect behavior is safe.
- [ ] Missed events have HTTP/query reconciliation.
- [ ] Stuck/missed operation detection exists for critical workflows.

## F. Data + deployment

- [ ] Staging/non-dev deployment proven.
- [ ] Production configuration separation proven.
- [ ] Secrets storage/rotation/incident process proven.
- [ ] Prisma migration on populated data rehearsed.
- [ ] Expand/backfill/contract strategy used for breaking schema changes.
- [ ] Code rollback proven.
- [ ] Migration rollback/recovery or forward-fix strategy proven.

## G. Operations

- [ ] Structured logs/correlation IDs.
- [ ] Error-rate and latency visibility.
- [ ] Financial anomaly/reconciliation alerts.
- [ ] Worker crash/restart recovery.
- [ ] Health/readiness semantics verified.

## H. Load

- [ ] 100 concurrent checkout scenario.
- [ ] 10 concurrent admin approval/release scenario.
- [ ] 500 concurrent socket connections.
- [ ] Retry storm/ambiguous provider scenario.

## I. UX/accessibility

- [ ] Loading/empty/error/success states.
- [ ] Keyboard/focus/screen-reader semantics.
- [ ] Reduced motion.
- [ ] Touch targets.
- [ ] Destructive confirmation.
- [ ] Optimistic update rollback.

## J. Final journey sign-off

- [ ] Auth/session.
- [ ] Marketplace checkout.
- [ ] Escrow/ticket workspace.
- [ ] Restaurant/dine-in.
- [ ] Hotel booking/check-in.
- [ ] Transit booking.
- [ ] Withdrawal.
- [ ] Business OS employee/EWA/payroll.
- [ ] Admin governance/financial workflows.

**Release decision:** PASS only when all P0 gates have evidence and all unresolved risks are explicitly accepted by the release owner.
