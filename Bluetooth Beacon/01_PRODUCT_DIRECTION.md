# 01 — Product direction

## Definition

**Azaman Presence** is a proximity-based discovery and interaction layer for Azaman's physical merchant marketplace. A participating business places a registered BLE beacon at an approved branch or zone. An opted-in customer with Azaman installed may detect the broadcast. The app submits an authenticated resolution request; the backend maps the device to its current branch assignment and approved merchant experience.

The purpose is to reduce manual search for participating nearby businesses.

## Customer proposition

A customer approaches a location and receives a useful prompt: “You're near [Business]. Open its Azaman storefront?” If several locations are plausible, “Choose the business you're visiting.” The prompt identifies the business and action. It never claims a purchase, reservation, check-in, pump selection or payment happened merely because a beacon was detected.

## Merchant experience modes

**Payment-only / digital storefront:** approved profile, location details and supported payment entry point, only when the backend capability exists.

**Catalogue / service experience:** restaurant menu, retail catalogue, hotel guest services or transit operator information. Ordering, booking, ticketing and payment remain separate operations under existing backend contracts.

A merchant may be discoverable without a physical beacon. Presence enhances marketplace discovery; it is not a prerequisite for a digital listing.

## Five verticals

| Vertical | Initial presence purpose | Later capability subject to separate contracts |
|---|---|---|
| Retail | Identify store and open catalogue/profile | Offers, cart, checkout, loyalty |
| Restaurants | Identify restaurant/approved dining zone and open menu | Orders, table service, split bill, payment |
| Hotels | Identify property/guest-service zone | Reservations, guest services, access-control integration |
| Fuel stations | Identify approved station; customer selects pump | Payment, POS/attendant verification, dispense, reconciliation |
| Transit | Identify operator/stop/station context | Routes, tickets, validator integration |

## Core principles

1. Reuse existing merchant, branch, tenant and permission model; no parallel merchant registry.
2. Beacon is a hint, not identity proof, exact distance, visit proof or payment authority.
3. Ask customer to choose when ambiguity exists.
4. Discovery, notifications and personalisation are separate opt-ins; direct marketing is separately consented and reviewed.
5. Merchants configure approved versioned templates, not arbitrary code or payment endpoints.
6. Only authoritative systems report completed transactions.
7. Prefer in-app cards while Azaman is open; system notification only when OS/user permissions permit.
8. Without gateway/vendor service, show last observation/test and age—not fake online/battery state.
9. Store only data required for specified purposes and retention.
10. Android/iOS background behaviour differs; do not guarantee universal delivery.

## Presentation contract

Merchant may configure IN_APP_ONLY or IN_APP_AND_NOTIFICATION. Effective behaviour intersects merchant setting, customer vertical choice, notification permission, app state, OS delivery, cooldown and ambiguity.

| State | Expected behaviour |
|---|---|
| Foreground | Discreet in-app card/sheet after resolution/preferences |
| Background; OS delivers event | System notification may appear if permission/policy allow |
| Notification tapped | Open Azaman, refresh context and resolve current approved route |
| Notification denied | No system notification; foreground still works |
| Bluetooth/permission unavailable | Honest fallback; marketplace search/map remains |
| App not installed | Beacon cannot create Azaman notification; use QR/NFC/signage/referrals |

An in-app modal cannot be drawn over another app or lock screen as general background behaviour. “Both” means foreground in-app card plus notification when background delivery occurs—not guaranteed simultaneous display in every state.

## MVP

Device registry; branch/zone assignment history; approved feature templates; supported Android/iOS detection; authenticated backend resolution; customer preferences/cooldowns; multi-store disambiguation; distinct customer observation and worker-test telemetry; basic admin visibility; tests for permissions, background delivery, dedupe, revocation and network failure.

Live payment execution is not part of the first radio/proximity proof.

## Out of scope until separately approved

Bulk rollout; pump inference/dispensing authority; digital room keys without access-control partner and security review; transit fare validation without operator integration; silent marketing from search/visit history; arbitrary merchant webviews/scripts/payment links; continuous health monitoring without telemetry.

## Success measures

Correct branch resolution, false positives, adjacent-business ambiguity, opt-in/out rates, prompt usefulness, merchant maintenance, support burden, battery impact and authoritative orders/reservations/payments linked to sessions.

Do not call Azaman detections “store footfall”. They represent opted-in Azaman devices/sessions observed—not everyone entering the premises.
