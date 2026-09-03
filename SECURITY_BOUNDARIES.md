# AZAMAN Security Boundaries

## Fundamental rules

- Authentication identifies an actor; authorization decides what the actor may do.
- A globally unique ID is never sufficient authorization.
- Business/tenant scope must be derived from trusted request context and/or proven object ownership.
- Privileged admin actions require explicit authority and auditability.
- Client-supplied business IDs cannot silently override trusted context.

## Business OS actor model

For each operation establish:

`actor → role/permission → business scope → target ownership → lifecycle state → side effects`

Known permission families include payroll view/process/disburse, EWA manage/approve and shift permissions. Route-level permission and service-level authorization must be intentionally consistent; service checks remain necessary defense in depth.

## Immediate boundary audits

- `ShiftService.updateShift()` must reject direct generic status mutation.
- `POST /shifts/rotation` must have intentional permission parity with shift creation.
- Attendance endpoints (`clock-in`, `clock-out`, `no-show`) need route/service authorization semantics documented and tested.
- EWA withdrawal must preserve actor self/admin/owner/authorized-employee rules and business scoping.
- Remaining invoices/orders/reservations/locations/products/reports/notifications must be mapped for IDOR risk.

## Admin security

Sensitive actions include withdrawal approval/rejection, escrow force-cancel/release, fee configuration and War Room actions. Each must have:

- explicit permission;
- target scope validation;
- lifecycle-state validation;
- transaction/idempotency protection;
- audit event without secrets/unnecessary sensitive data;
- deterministic response under replay/concurrency.

## Secrets

Production readiness requires evidence for storage, access, rotation and incident response for database credentials, JWT signing material, payment/MoMo provider credentials, FCM/server keys and other secrets. Never put secrets in source, logs, events or planning documents.

## Red-team cases

- cross-business target IDs;
- privilege escalation;
- suspended/inactive staff continuing access;
- replayed financial/admin requests;
- generic PATCH state manipulation;
- forged/overbroad business context;
- sensitive data leakage through errors/events/audit logs.
