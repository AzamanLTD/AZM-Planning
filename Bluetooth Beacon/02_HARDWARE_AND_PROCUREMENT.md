# 02 — Hardware and procurement

## Procurement position

Hardware is **not selected**. The research shortlist contains:
- **Minew MBM02:** candidate battery-powered beacon.
- **Minew C3:** candidate compact USB-powered beacon.

Supplier specifications are not independent performance guarantees or Ghana regulatory approval. Confirm exact revision, firmware, certificates, battery/accessory configuration and NCA status before purchase.

No bulk order is recommended. A small prototype quantity (for example two of each) may be considered after documentation/compliance checks. Fuel-station devices are excluded from the first sample deployment.

## Candidate comparison

| Candidate | Supplier-stated characteristics | Questions to close |
|---|---|---|
| Minew MBM02 | BLE 5.0; 4,000 mAh lithium battery; up to 150 m stated open-area range; IP67; advertised up to 10 years under default settings | Exact revision; battery chemistry/replacement; measured life at chosen interval/power; temperature; battery telemetry; secure provisioning; mounting; certificates; NCA status |
| Minew C3 | USB-powered compact BLE 5.0; supplier lists iBeacon/Eddystone; approx. 18 × 14 × 6 mm; up to 50 m stated range | USB supply; brownout/restart; firmware updates; secure identity; current draw; telemetry; mounting; exact NCA status |

Official pages:
- MBM02: https://www.minew.com/product/mbm02-proximity-navigation-beacon/
- C3: https://www.minew.com/product/c3-usb-ble-beacon/

Do not assume battery percentage is advertised because a device supports iBeacon. Ask whether the exact firmware supports documented telemetry and how it is exposed.

## Supplier RFI

Obtain written answers per exact model/revision.

**Radio:** BLE version/formats; iBeacon identifier configurability; persistence after power loss; advertising interval/transmit-power settings; indoor detection evidence; interference and mobile background limitations.

**Identity/security:** serial/manufacturing identity; secure provisioning/ownership transfer/decommission; vendor-backed rotating identifiers; per-device key injection/storage/rotation/revocation; signed firmware updates/rollback; security support lifecycle. “Encrypted data” is insufficient without protocol/threat model/server resolution. Static UUID/Major/Minor can be copied.

**Power/maintenance:** battery/USB characteristics; measured life at Azaman settings and representative temperature; replacement; voltage/health telemetry; whether remote health requires gateway/vendor subscription/nearby phone; low-battery and USB-loss behaviour.

**Physical/regulatory:** dimensions, IP rating, operating temperature, mount; declarations/radio reports; NCA approval status; warranty/replacement lead time/region/MOQ/pricing. Fuel-site use additionally requires hazardous-area suitability evidence and site approval. IP67 is not intrinsically safe or explosion-proof certification.

## Ghana equipment checks

Confirm exact manufacturer, brand, product type and model against current NCA type-approval requirements. Do not infer Ghana approval from CE/FCC marks or supplier statements.

- NCA FAQs: https://nca.org.gh/faqs/
- NCA Type Approval & Conformance: https://nca.org.gh/type-approval-conformance/

Confirm applicable route with NCA or qualified local adviser.

## Provisioning and commissioning

1. Receive device; record make/model/revision/serial/condition.
2. Confirm approval/certification and firmware.
3. Allocate Azaman device ID.
4. Provision radio identity using approved vendor tooling.
5. Register identity server-side; no fleet-wide Flutter secret.
6. Assign to existing branch and approved zone.
7. Tune transmit power/interval for smallest useful area.
8. Install only at site-approved location and record placement.
9. Test representative Android/iOS devices.
10. Record worker evidence; activate only after validation.
11. Audit reassignment, repair, replacement and decommission.

Do not reuse identifiers after loss/decommission without revocation/reassignment process.

## Base deployment

Standalone BLE: beacon broadcasts locally; phone detects; phone uses its own data/Wi-Fi to contact Azaman. Beacon need not authenticate users or connect to the internet. This avoids merchant Wi-Fi/Ethernet/gateway/SIM for discovery, but means no continuous remote health. Gateway/vendor service is separate architecture/cost.

## Gates

Before samples: current datasheets, NCA status, identity/provisioning docs, telemetry claims, security options, price/warranty and non-fuel install locations.

Before bulk order: controlled field pilot, quantified coverage/false positives/maintenance/customer response, documented budget and commercial approval.

**No bulk purchase or fuel-site deployment is approved by this plan.**
