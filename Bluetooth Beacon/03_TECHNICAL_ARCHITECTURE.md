# 03 — Technical architecture

## System context

Use Azaman's existing Flutter app, JavaScript/Node.js backend, PostgreSQL database and Vite merchant portals. Presence is a reusable module, not a parallel merchant system or financial authority.

**Flow:** registered BLE beacon → customer phone → authenticated Azaman API → existing merchant/branch resolution → approved feature template → Flutter storefront route.

The beacon is a broadcast-only proximity hint. The customer's phone supplies connectivity. Backend remains authoritative for merchant identity, device assignment, permissions, feature publication and transaction state.

## Responsibilities

### Flutter
Permission explanations; native-capable detection; short-window client dedupe; authenticated observations; backend resolution; preferences/cooldowns; foreground card; notification-tap routing; typed template models and existing route registry. No fleet secret or arbitrary merchant action URL.

### Native mobile
- Android: evaluate filtered BLE scanning with BluetoothLeScanner.startScan and PendingIntent for background events subject to current OS/device policy.
- iOS: evaluate Core Location iBeacon region monitoring/ranging subject to authorisation/background/OS behaviour.
- Provide typed bridge to Flutter.
- Do not assume periodic WorkManager is a continuous scanner.
- Test force-stop, termination, battery restrictions and revoked permission.

### Node.js/Express
Authenticate customer/merchant/worker; resolve registered device and assignment; enforce tenancy/branch/device/config; accept deduplicated observations and bounded sessions; apply preferences/cooldowns; serve typed versioned templates; record audit/operational evidence. Presence does not create payment/order/settlement.

### PostgreSQL/Prisma
Conceptual records to evaluate against existing schema: device inventory; identifier/provisioning metadata; branch/zone assignment history; versioned feature configs; user preferences; deduplicated observations/sessions; worker tests/commissioning; health samples only when supported; audit records.

These are concepts, not table-creation instructions. Inspect existing BusinessProfile, branch, staff, permission, audit and feature models first. Reuse tenant identity/audit primitives and existing migrations/deployment workflow.

## API capabilities

Endpoint names await route-registry inspection.

| Capability | Caller | Server responsibility |
|---|---|---|
| Resolve/submit observation | Authenticated Flutter | Derive user from session; resolve active device/branch; eligibility/dedupe |
| Fetch feature | Flutter | Return constrained versioned experience |
| List/assign/revoke devices | Merchant staff | Tenant/branch check; preserve history |
| Save experience | Merchant staff | Validate template; audit |
| Worker test | Assigned worker | Validate assignment; separate test evidence |
| Fleet map | Admin | Return assignments, observation/test/health evidence |

Never trust request-supplied userId or merchantId as authority. Resolve principal and scope from authentication and permissions.

## Resolution response

Return device ID/state/zone, existing merchant and branch identity, vertical, approved template key/schema version, effective presentation mode, cooldown, internal allowlisted route ID/typed parameters and correlation ID.

Do not return executable JS, arbitrary external redirects, provider secrets, unnecessary raw DB IDs or arbitrary UI definitions.

## Device identity

Static iBeacon tuple (UUID/Major/Minor) can be cloned. It may support controlled functional prototyping but is not authentication. Design for manufacturer serial, Azaman device ID, identity generation, registration/revocation, optional vendor-backed rotation and server-side resolution.

Do not implement HMAC with a fleet secret embedded in Flutter. App secrets can be extracted; a signature proves possession of copied app key, not physical beacon presence. Relay remains possible.

## Presence session model

A scan callback is not a visit. Repeated advertisements and RSSI fluctuation are expected.

1. Receive observations for an active device.
2. Validate device and authenticated user.
3. Deduplicate bursts.
4. Use stable criteria/hysteresis to open a candidate session.
5. Close after absence threshold, not one missed packet.
6. Apply user/device/branch cooldown.
7. Count later session only after meaningful separation.
8. Link storefront/selection and separately recorded backend conversion.

Do not increment visits on every callback. Define and measure qualified-session thresholds.

## Multiple locations

A branch may have zones; one building may contain multiple merchants. Strongest RSSI is not definitive; RSSI/GPS cannot reliably infer floor or pump. If several candidates plausible, group by existing identities, show customer choice, use labels only when configured and record ambiguity.

## Network failure and offline

Live authenticated resolution is needed to verify current assignment and feature. On outage: don't open stale transaction-capable screen; don't count a delayed signal as current visit; any diagnostic queue needs purpose/expiry/replay/privacy definition; no offline presence authorises payment/ticket/room key/pump/order/settlement. Provide retry/search fallback.

## Observability and financial separation

Use correlation IDs; server receipt time is authoritative; client timestamp untrusted. Collect only signal metadata needed to tune/diagnose. Presence is navigation. Orders/bookings/tickets/payments use existing authorities. GHS and USDC stay distinct; display valuation is not spendable funding; beacon never proves GHS liquidity or resolves unknown provider outcome.
