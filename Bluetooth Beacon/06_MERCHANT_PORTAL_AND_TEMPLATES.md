# 06 — Merchant portals and approved feature templates

## Purpose

Vite portals provide controlled management of the merchant's own devices and storefront config. They are not a second merchant database or arbitrary remote-execution system in Flutter. Node backend remains authority for tenant/branch permissions, validation and publication.

## Merchant setup

1. Merchant has approved existing profile/branch.
2. Permitted staff user opens Presence portal.
3. Portal shows devices assigned/available to authorised branch.
4. Merchant requests/confirms assignment under role policy.
5. Merchant selects approved vertical template.
6. Portal exposes only fields allowed by template.
7. Backend validates, versions and audits actor.
8. Worker/operator runs commissioning test.
9. Publish only after assignment/schema validation.

Merchant cannot claim arbitrary beacon IDs or move device to another business without approved transfer.

## Versioned templates

| Template key | Vertical | Example fields |
|---|---|---|
| retail.catalogue | Retail | Existing catalogue/profile ref, approved offers, labels |
| restaurant.menu | Restaurant | Existing menu ref, service mode, opening details |
| hotel.guest_services | Hotel | Approved service list, property profile, contacts |
| fuel.station_payment | Fuel | Station identity, pump list, workflow ref—disabled until fuel gates clear |
| transit.operator_info | Transit | Operator, stop/station ref, route/ticket info |

Each template defines version, vertical, required/optional fields and bounds, allowed route ID, permission/publication conditions, fallback and deprecation/migration.

## No arbitrary execution

Reject executable JS/scripts, arbitrary WebView URL/redirect/scheme, provider endpoint/credentials, tenant-bypass IDs and arbitrary native/transaction actions. Flutter maps known key/version to reviewed renderer; payload is typed data. Orders/bookings/payments use existing authorised endpoints.

## Device management fields

Device ID, make/model/serial/provisioning state; branch/zone and assignment date; commissioning result; last customer observation; last worker test; health only if supported; firmware/config version where known; state such as uncommissioned, active, due, issue, suspended, revoked, decommissioned.

Issue/reassignment actions are role-controlled and audited.

## Publish lifecycle

Draft is not active. Backend validates schema and permission. Published version has actor/time/immutable ID. Mobile resolves only active version. Rollback is audited. Revocation blocks resolution. History retained under policy.

## Portal/API security

Vite calls authenticated routes. Server derives staff/tenant scope from session and existing permission logic. Never trust browser merchantId/branchId alone. Validate payload size/keys/types/lengths/references/template version.

## Merchant analytics

Report Azaman opted-in observations/qualified sessions, cards shown/opened, user selections, backend-confirmed orders/reservations/payments, worker verification and last observed/tested time.

Do not report total footfall or client-reported conversions. No gateway means no continuous online state; people without app/permission are not visible.

## Acceptance

Tenant isolation; role restrictions; schema validation; arbitrary URL/script/payment endpoint rejected; draft not customer-resolved; publication/rollback audited; revoked device stopped; observation/test labels accurate; no fabricated battery state.
