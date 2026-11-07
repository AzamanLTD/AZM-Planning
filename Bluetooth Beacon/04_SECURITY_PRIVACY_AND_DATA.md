# 04 — Security, privacy and data protection

## Threat model

Assume nearby attackers can copy static identifiers, broadcast fake identifiers, relay genuine signals, replay events, alter timestamps, call APIs directly, attempt cross-tenant access or extract app-embedded secrets.

## Beacon is not transaction authority

Static iBeacon identifier is a routing hint, not proof that the physical device is genuine or a person is standing there. Secure rotating identity may reduce casual cloning only if exact firmware supports it, keys are unique per device, provisioning/rotation/backend resolution/revocation are documented and vendor support is sustainable. Cryptographic identity does not eliminate relay attacks.

**Never authorise payment, fuel dispensing, room access, ticket validation or another sensitive action solely from beacon presence.**

## Authentication/authorisation

- Customer identity comes from authenticated session; client userId is not authority.
- Merchant/staff scope comes from authenticated principal and permission model.
- Worker must be assigned to branch/device and hold correct role.
- Admin global inventory/map requires explicit permission.
- Validate device active/revoked state and assignment at server receipt.
- Server generates durable IDs/timestamps; client time is untrusted context.
- Rate-limit APIs; bound/validate payloads.
- Audit provisioning, assignment, revocation, configuration and worker tests.

## No fleet secret in Flutter

A shared HMAC secret in APK/IPA can be extracted. Signing a timestamped tuple with it does not prove the beacon emitted the signal. Avoid shared secrets in Dart/native/assets/remote config and public identifier-derived secrets. If vendor identity uses keys, use per-device credentials in protected server-side key management and define provisioning, rotation, revocation, audit and recovery.

## Separate customer controls

1. **Proximity discovery:** permission to detect opted-in locations where OS allows.
2. **System notifications:** permission for alerts while app is not active.
3. **Personalisation:** permission to use allowed aggregates/search-interest to rank merchants.
4. **Direct marketing:** separate consent/controls subject to Ghana requirements.

Selecting “restaurants” is not blanket consent to background notification, detailed visit history or marketing. Provide per-vertical controls, global pause/disable and clear revocation. Declining optional discovery/marketing should not disable unrelated app features.

## Data minimisation

- Store qualified sessions, not every BLE packet.
- One RSSI observation is not a visit.
- Do not retain detailed movement history when aggregate count is enough.
- Do not turn search history into automatic interruptive marketing.
- Prefer in-app recommendation where opted in.
- Avoid precise coordinates when branch/zone is enough.
- Define retention for raw events, sessions, aggregates, diagnostics and audit evidence.
- Provide applicable access/correction/objection/withdrawal/deletion processes.
- Pseudonymise/aggregate where identity isn't needed.

“Two visits” is an experiment, not a permanent universal rule. Evaluate only on deduplicated sessions with retention defined.

## Ghana privacy review

Before live tracking/personalisation:
- Confirm controller/processor roles for Azaman and merchants.
- Confirm DPC registration/compliance obligations.
- Document purposes, data categories, recipients, retention, access and notices.
- Assess need for privacy impact assessment.
- Obtain valid consent where required, including prior written consent for direct marketing where applicable.
- Provide practical marketing stop and discovery disable controls.

References:
- DPC compliance: https://dpc.gov.gh/compliance/
- Data Protection Act 2012 (Act 843): https://ghalii.org/akn/gh/act/2012/843/eng%402012-05-18

This is not legal advice; obtain Ghana-qualified privacy counsel before live processing.

## Merchant content and endpoint safety

Use closed versioned typed templates. Validate payload, tenant/branch ownership and capability. Allow only internal allowlisted Azaman route IDs. Reject executable HTML/JS, arbitrary WebView URLs, redirects, scripts, payment endpoints and native actions. Merchant data cannot select provider credentials or alter money movement.

## Payment separation

Presence can open an existing workflow; cannot confirm it. Fuel customer explicitly selects pump; authorised POS/attendant/forecourt verifies payment and dispense. Screenshot/client boolean is not settlement evidence.

GHS and USDC remain distinct. Displayed GHS equivalent of USDC is not spendable GHS. Presence cannot create liquidity/customer funds.

## Security acceptance

Unknown/revoked ID fails; static clone has no privilege; supplied user/merchant ID cannot impersonate; tenant isolation holds; replay is deduped; rotating identity failure is closed; arbitrary URL/code/action rejected; rate/size bounds hold; logs contain no secrets/unneeded history; preferences independently enforced; presence cannot debit, order, authorise fuel or alter payment result.
