# AZAMAN Living Architecture

## System responsibility

AZAMAN is one platform, not five independent products.

**Identity & Trust → Unified Money/Ledger → Domain State → Experience/SDUI → Realtime/Eventing → Notifications/Social → Observability/Reconciliation**

| Repository | Authority / responsibility |
|---|---|
| `AZM-backend` | Server authority: identity, money, domain state, authorization, APIs, events |
| `AZM-frontend` | Consumer experience; renders authoritative backend outcomes |
| `AZM-businessPortal` | Business operating/control surface; configures business-owned state |
| `AZM-adminPortal` | Governance/control plane; privileged operations must remain explicitly authorized/audited |
| `AZM-Planning` | Engineering memory, contracts, roadmap and release evidence |

## Authority rule

Backend owns persisted business and financial truth. Portals/apps may cache, estimate or optimistically render, but must reconcile to committed server state. Realtime signals accelerate convergence; they never become a second source of truth.

## Vertical model

Shared primitives should serve Retail, Restaurant, Hotel and Transit rather than cloning independent payment/order/inventory implementations. Category-native UX is allowed; duplicated authority is not.

## Experience Blueprint

**Business Portal configuration → Backend validation/storage/API → Flutter rendering.**

Blueprint is a first-class architecture surface and must appear in cross-repo contract audits.

## Critical transaction pattern

For financial/stateful mutations:

`request context → authorization → tenant/object scope → idempotency/claim → transaction/conditional transition → ledger/balance/domain effects → committed event → client reconciliation`

Events must not announce an uncommitted state. If event delivery fails, a durable/recoverable reconciliation mechanism must eventually restore client convergence.

## Cross-service failure model

The backend database transaction is authoritative. Socket delivery, push notification, portal query invalidation and mobile UI updates are downstream effects. Design every important journey so a client can recover after:

- socket disconnect;
- event loss;
- duplicate event;
- request timeout after server commit;
- stale cached query;
- provider response ambiguity.

## Architecture anti-patterns

- client-calculated authoritative money;
- generic PATCH endpoints that bypass lifecycle side effects;
- authorization inferred from globally unique IDs;
- duplicated service implementations for compatibility;
- local state treated as proof of committed backend state;
- socket event treated as durable truth;
- static/demo fallback silently replacing failed authoritative data;
- parallel PRs implementing the same logical fix.
