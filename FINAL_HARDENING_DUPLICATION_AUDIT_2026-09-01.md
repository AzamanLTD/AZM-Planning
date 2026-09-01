# Final Five-Repo Duplication Audit — 2026-09-01

## Scope

Repositories audited:

- `AZM-backend`
- `AZM-adminPortal`
- `AZM-businessPortal`
- `AZM-frontend`
- `AZM-Planning` as the architecture record and cross-check source

Focus:

- duplicate API wrappers;
- duplicate realtime listeners;
- duplicate event constants;
- competing financial implementations;
- stale compatibility layers;
- contradictory status transitions.

No consolidation is proposed without proof of canonical ownership.

## Findings

### 1. Financial transport wrappers — intentional boundary, not duplicate

**Backend / Admin Portal:** `financialApi` is the validated financial facade; generic `api` remains the transport for unaudited or non-financial Admin endpoints. Shared settings, fees, disputes, withdrawals, payouts and escrow mutations use the financial facade where the audited response boundary exists.

Decision: **DO NOT CONSOLIDATE.** The two layers have different responsibilities. Replacing every generic call would widen the financial facade without an established contract and would recreate transport duplication inside the facade.

### 2. Realtime connection ownership — canonical singleton/bridge

**Flutter:** `lib/services/socket_service.dart` is the singleton Socket.IO owner and central listener registry. Marketplace/ticket/order consumers register callbacks instead of creating competing global sockets.

**Business Portal:** the existing realtime bridge is the canonical consumer projection boundary; socket events invalidate/refetch canonical query roots rather than maintaining a parallel authoritative state.

**Admin Portal:** `useAdminRealtime.js` is the global invalidation boundary. Escrow and withdrawal alerts converge into query invalidation rather than maintaining local financial state.

Decision: **DO NOT CONSOLIDATE.** These are separate surface-specific consumer boundaries sharing one architectural pattern, not duplicate socket transports.

### 3. Financial provider implementations — complementary, not competing

Backend payment failover keeps Moolre as primary and MTN as secondary through the existing `PaymentFailoverService`. The individual provider services are I/O adapters; the failover service owns provider selection/health.

Decision: **DO NOT CONSOLIDATE.** Removing either adapter would remove an intended provider role. Adding a second failover implementation would be the actual duplication risk.

### 4. Admin alert rooms — deliberate topology split

Generic `AdminAlertService` uses `admin_spy_room` as its operational projection boundary. Withdrawal settlement uses `admin_spy` for its specific `admin_alert` topology, while the user-facing withdrawal projections remain `user_<userId>`.

Decision: **DO NOT CONSOLIDATE.** The different room names are part of separate producer/consumer contracts and are now executable-test covered.

### 5. Event names — canonical producer/consumer strings remain distributed by surface

The same event strings appear in Backend producers and the three clients because Socket.IO is a wire protocol, not a shared source-code module. The duplication is structural contract representation, not duplicate implementation.

Current executable Backend contract coverage locks:

- `escrow_funded`
- `escrow_settled`
- `escrow_pending_settlement`
- `escrow_refunded`
- `invoice_paid`
- `withdrawal_progress`
- `withdrawal_settled`
- `admin_alert`
- `order.delivered`
- `order.completed`

Decision: **KEEP.** Centralize by documented contract/tests rather than forcing a cross-repository shared library.

### 6. Order state machines — Backend owns the authoritative transition

`BusinessOrderStatus` is defined by the Prisma model and the mutation services/controllers enforce conditional transitions. Business/Flutter/Admin consumers project those statuses but do not define a second financial state machine.

Decision: **KEEP BACKEND AUTHORITY.** Consumer-side status displays are projections, not competing transition engines.

### 7. Marketplace API access — evidence of canonical backend capability

The Backend `/api/business` router owns the marketplace business/order/product/invoice/review/location/reservation/transit capabilities. Flutter and Business Portal use consumer-specific clients/services to reach those routes.

Decision: **NO duplicate backend capability found.** Consumer wrappers should remain thin and surface-specific.

### 8. Compatibility layers

Known compatibility behavior includes the historical `mtnDisbursementService` key pointing at the primary Moolre disbursement service while the failover service retains an explicit Moolre→MTN provider order. This is intentionally preserved because other parts of the application still consume the historical key.

Decision: **KEEP until all consumers can be migrated atomically.** Renaming it opportunistically would create a larger cross-repository migration risk than the current compatibility alias.

## High-risk duplicate patterns to prohibit going forward

1. A second Admin financial transport outside `financialApi` for an already-audited mutation.
2. A second global Flutter Socket.IO instance or second Business Portal realtime bridge.
3. Any direct balance/ledger mutation in a consumer application.
4. Any new backend provider-failover implementation outside `PaymentFailoverService`.
5. Any second status-transition engine for `SmartEscrow`, `BusinessOrder`, withdrawal settlement, or invoice payment.

## Conclusion

The deep audit found **architectural repetition but no proven duplicate implementation that should be removed now**. The repeated pieces are primarily contract projections or deliberate compatibility boundaries. The safest consolidation strategy is therefore continued executable contract coverage plus targeted source cleanup only when a reachability proof establishes a genuinely superseded implementation.

This document is evidence for the final hardening gate; no source deletion or cross-repository refactor is bundled with it.
