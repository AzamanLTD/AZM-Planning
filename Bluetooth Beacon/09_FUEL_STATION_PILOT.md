# 09 — Fuel-station pilot: separate safety and payment gates

## Why separate

Fuel may be strategically valuable but has greater safety, operating and payment dependencies than retail/restaurant discovery. BLE cannot establish whether phone use is allowed at a forecourt point, whether a device is suitable, which pump is used or whether fuel was dispensed.

**Do not make fuel the first proof of the general proximity network.** Start retail/restaurant field trials and work on fuel partner requirements in parallel.

## Safety first

Before installation or customer phone workflow:
- Obtain written OMC/station operator and competent site safety approval.
- Determine exact hazardous-area classification and permitted location.
- Verify exact beacon model/revision and required hazardous-area certification/assessment. IP67 alone is not intrinsically safe or explosion-proof.
- Confirm BLE transmission and customer phone use permitted at precise location and under site rules.
- Confirm mounting, maintenance, battery, cabling and worker procedures.
- Do not assume canopy, wall, office, entrance or supervisor booth is outside a restricted zone.
- If phone use is not permitted where interaction is needed, do not prompt there; redesign around approved safe interaction point or stop.

No installation is safe based only on illustration, supplier marketing or IP rating.

## Station is not pump

Beacon identifies approved station/zone. RSSI varies with structures, metal vehicles, interference, people, orientation and neighbouring transmitters. It is not reliable pump selection.

Customer explicitly selects/confirms station, pump, fuel grade, amount/quantity and final summary. Station/attendant/POS/forecourt independently verifies pump and payment.

## Future transaction journey

1. Resolve approved station and display identity.
2. Customer manually selects pump from published station config.
3. Customer selects grade/amount; show summary.
4. Existing Azaman backend creates idempotent payment intent and determines genuine source asset/approved route.
5. Provider returns final state or unknown outcome held for reconciliation.
6. Authorised POS/attendant/forecourt verifies reference, amount and pump.
7. Fuel dispensed under station process; record actual quantity when integration supports.
8. No/partial/over/under dispense, cancellation and unused preauthorised funds follow defined adjustment/refund rules.
9. Customer receives final authoritative status/receipt.

Screenshot/client success flag is not evidence.

## Money/settlement boundaries

- Use only actual spendable GHS with custody/funding evidence and product authority.
- USDC or displayed GHS equivalent is not spendable GHS.
- Never create synthetic GHS/treasury opening to appear funded.
- Beacon cannot debit wallet, reserve, select provider, dispatch or mark success.
- Provider uncertainty stays unresolved; no blind retry/provider switching.
- Contract covers finality, reversal, refund, idempotency, reconciliation, settlement account and commercial responsibility.
- OMC confirms settlement/reconciliation.
- Do not claim any provider supports this workflow without specific contract and working evidence.

## Partner checklist

OMC/station approval; site safety review and signed installation/phone-use procedure; exact location; NCA type approval; hazardous-area evidence if applicable; forecourt/POS interface; pump/grade catalogue; provider/acquirer and settlement design; attendant training/support; mismatch reconciliation; incident/device failure/removal plan.

## Go/no-go

Start with non-transactional station info only after safety/partner permission. Enable payment only after safety, provider, GHS source, POS/attendant, dispense, refund and reconciliation are approved/tested.

No-go if model suitability or phone-use/location approval unknown; station cannot verify payment; RSSI infers pump; GHS source unapproved; provider finality unknown; dispense reconciliation undefined; or success shown before authoritative confirmation.

Fuel pilot is a controlled partner programme, not automatic extension of retail proximity.
