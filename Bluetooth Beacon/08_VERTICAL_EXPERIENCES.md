# 08 — Five marketplace vertical experiences

## Shared layer

All verticals use registered device/branch mapping, authenticated resolution, preferences/cooldowns, approved template/version, customer choice where ambiguous, and existing order/booking/payment/ticketing authority after storefront opens. Beacon does not determine the transaction.

## Retail

**Prompt:** “You're near [Retailer]. Browse its catalogue?” Initial: store profile/location, catalogue, active approved offers, hours/contact. Later: availability, cart, checkout, loyalty/pickup through authoritative inventory/order/payment. Do not show expired or branch-inapplicable offers.

## Restaurants

**Prompt:** “You're near [Restaurant]. View the menu?” Initial: identity, menu, service information; optional dining-zone context only when configured/reliable; ask user when uncertain. Later: ordering/table service/split bill/tips/payment through existing backend authority. Beacon is not an order or proof of table occupancy/intent.

## Hotels

**Prompt:** “You're at [Hotel]. Open guest services?” Initial: property profile, approved services, reservation/arrival info and support contact. Later: booking changes/guest requests/access control. Digital key/unlock requires separate access-control partner, strong identity, device security, revocation and lockout review. Beacon alone cannot unlock.

## Fuel

**Prompt:** “You're near [Station]. Open station details?” First release identifies approved station only. Customer explicitly selects pump, fuel grade and amount; RSSI never infers pump. Future flow requires site approval, real GHS funding, idempotent payment intent, authorised payment, POS/attendant confirmation, dispense/reconciliation and final receipt. See [09 — Fuel station pilot](09_FUEL_STATION_PILOT.md).

## Transit

**Prompt:** “You're near [Transit Operator/Stop]. View routes?” Initial: operator/station/stop, published route/timetable/service info and approved ticket path. Later: ticket issue/fare/seat/validation via operator/validator and server-authoritative ticket state. Beacon is not ticket validator or proof of boarding.

## Template contract and rollout

Each vertical uses fixed versioned template/schema and reviewed Flutter renderer. Backend returns allowlisted route ID and validated data, not arbitrary components/actions.

Start with **retail and restaurants** for the general proof. Hotels/transit follow when service APIs/partners are ready. Fuel is separately gated with an OMC/site partner.
