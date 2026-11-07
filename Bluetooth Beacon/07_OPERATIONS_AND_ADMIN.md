# 07 — Operations and administration

## Operating model

Separate device registration, commissioning, assignment, worker tests, customer observations and supported health telemetry. They are distinct evidence types, not one “online” flag.

## Device lifecycle

Received/inventory pending → provisioning → branch/zone assigned → commissioning → active → verification due/issue reported/suspended → replacement → revoked/decommissioned.

Every transition needs authorised actor, timestamp and reason. Preserve previous assignments so historical observations can be evaluated against valid assignment.

## Worker daily verification

Authorised branch worker selects **Test Azaman Presence**:
1. Authenticate worker and verify branch assignment.
2. Run time-bounded native scan for expected device.
3. Detect expected device or fail.
4. Check registration/assignment.
5. Check published template resolves.
6. Capture maintenance signal diagnostics, not exact-position claim.
7. Submit result with server time/correlation ID.
8. Show honest pass/fail and next action.

Worker test must not open customer storefront as a customer, notify customers, count customer visit, create order/booking/ticket/payment or alter config/assignment without separate role.

Use distinct WORKER_TEST_PASSED/FAILED evidence; commissioning is separate.

## Customer vs worker evidence

Customer observation, qualified session, storefront candidate, customer selection/open, backend-confirmed conversion, worker test, commissioning, vendor/gateway health sample (if supported) are separate records.

A customer observation only means an opted-in phone detected beacon at a moment. It is not a continuous heartbeat or guaranteed detectability.

## Device health/battery

Generic iBeacon does not guarantee battery telemetry. Eddystone-TLM/vendor data may expose voltage/temp/counters only if exact hardware/firmware supports.

Without gateway/vendor service:
- Last observation is not device health.
- No observation does not prove offline/dead battery.
- Never default battery to 100%.
- Use “not recently observed” or “health unknown”, not “offline”.
- Offer worker verification.
- Separate hardware telemetry from customer observation.

Gateway/vendor platform is separate cost/security/network/support decision.

## Admin map

Show existing branch/zone, device/model, assignment/history, commissioning evidence, last customer observation, last worker test, genuine telemetry source/time, issue, replacement/decommission. Filter by vertical, region, branch, model, assigned state, verification, issue, observation age and commissioning. Coordinates/floors from approved configuration, not RSSI. Admin-only.

## Marketplace map

Customer map uses existing merchant/branch source. It may show Presence availability, not imply recent observation unless meaningful. No duplicate merchant location store.

## Metrics

| Metric | Definition |
|---|---|
| Observation count | Server-accepted, deduped observations |
| Qualified sessions | Defined stable/separated sessions |
| Prompt rate | Eligible prompt actually shown |
| Selection rate | Customer-selected results / choices shown |
| Notification open | Tapped issued notifications |
| Confirmed conversion | Backend authoritative completed order/payment/reservation |
| Worker coverage | Devices with passing worker test in window |
| Device health | Vendor/gateway telemetry with source/time, else unknown |

Never call sessions footfall/unique visitors. Presence excludes people without app/Bluetooth/permission.

## Alerts and acceptance

Alert for worker failure, invalid assignment, unpublished config, supported telemetry threshold, lost/damaged/moved device or stale observation/test (label stale/unconfirmed, not offline). Transition-based alerts prevent repeats. Record source/severity/acknowledgement.

Every active device has branch/zone assignment and commissioning. Reassignment/revocation audited. Worker test separate. Admin statuses honest. Merchant access tenant-scoped. Analytics authoritative. Health claims evidenced.
