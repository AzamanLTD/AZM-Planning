# References and research provenance

**Research reconciliation:** 2026-11-07. Recheck current supplier and regulatory details before action. This is not legal or safety advice.

## Mobile platforms

### Android
- Background BLE scanning: https://developer.android.com/develop/connectivity/bluetooth/ble/background
- Bluetooth permissions: https://developer.android.com/develop/connectivity/bluetooth/bt-permissions
- BluetoothLeScanner API: https://developer.android.com/reference/android/bluetooth/le/BluetoothLeScanner

### Apple
- iBeacon proximity: https://developer.apple.com/documentation/corelocation/determining-the-proximity-to-an-ibeacon-device
- Core Location: https://developer.apple.com/documentation/corelocation

## Hardware vendors
- Minew MBM02: https://www.minew.com/product/mbm02-proximity-navigation-beacon/
- Minew C3: https://www.minew.com/product/c3-usb-ble-beacon/
- Minew beacon catalogue: https://www.minew.com/product-category/bluetooth-beacon/

Dimensions, battery capacity/life, IP rating and range are supplier claims. Confirm exact model/revision and test in environment.

## Ghana regulatory/privacy
- NCA FAQs: https://nca.org.gh/faqs/
- NCA Type Approval: https://nca.org.gh/type-approval-conformance/
- DPC compliance: https://dpc.gov.gh/compliance/
- Data Protection Act 2012 (Act 843): https://ghalii.org/akn/gh/act/2012/843/eng%402012-05-18

Verify exact manufacturer/brand/product/model with NCA. Foreign radio approval is not Ghana approval. Obtain local advice on registration, controller/processor, lawful basis, notice, consent, retention and direct marketing.

## Beacon protocol/security
- Google Eddystone repository (archived): https://github.com/google/eddystone
- Eddystone-TLM: https://github.com/google/eddystone/tree/master/eddystone-tlm

Do not assume archived repository has maintained turnkey cloud resolution.

## Research caveat

Original working research transcript contains early illustrative SQL, Node.js, Flutter, HMAC and UI snippets. They are brainstorming, not approved implementation. Later review identified problems: app-embedded shared secret, visit inflation from repeated scans, client-supplied identity, arbitrary merchant payload execution, optimistic background guarantees, unproven battery telemetry and pump inference. This folder supersedes those snippets.

## Image provenance

Six concept images under assets were generated for this Azaman Presence planning exercise and optimised for repository size/readability. They are illustrations, not manufacturer schematics, compliance evidence or photos of installed devices.
