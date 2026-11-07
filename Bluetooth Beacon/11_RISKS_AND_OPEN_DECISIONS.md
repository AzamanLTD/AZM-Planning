# 11 — Risks and open decisions

This register keeps unresolved product, regulatory, commercial and engineering questions visible. Agents must not invent answers.

| ID | Risk / decision | Current position | Evidence / owner action |
|---|---|---|---|
| BB-01 | Supplier/model | MBM02/C3 candidates only | RFI, exact revision, sample, support/cost |
| BB-02 | Ghana NCA approval | Not confirmed | Exact manufacturer/brand/type/model check |
| BB-03 | Secure rotating ID | Not confirmed for candidate | Protocol, provisioning, rotation, resolution, revocation |
| BB-04 | Clone/relay | Inherent risk | Signal hint only; no payment authority |
| BB-05 | Android background | OS/OEM dependent | Native physical-device test matrix |
| BB-06 | iOS background | Core Location/permission dependent | Background/force-quit/permission tests |
| BB-07 | Indoor/floor precision | RSSI not exact | Customer choice; no pump inference |
| BB-08 | Adjacent merchants | Expected ambiguity | Multi-candidate UX and field data |
| BB-09 | Tracking/retention | Privacy design required | Minimisation, retention, DPC/legal review |
| BB-10 | Direct marketing | Separate consent review | Explicit consent, opt-out, counsel |
| BB-11 | Repeat visits | Two visits is hypothesis | Session definition/cooldown/pilot |
| BB-12 | Battery telemetry | Unknown | Vendor proof or health unknown |
| BB-13 | No gateway | Accepted limitation | Later gateway/subscription decision |
| BB-14 | Merchant self-service | Not final | Roles, custody, assignment/publishing |
| BB-15 | Template ownership | Not final | Backend schemas and renderer lifecycle |
| BB-16 | Offline event | Cannot imply present/payment | Diagnostic queue purpose/integrity/expiry |
| BB-17 | App acquisition | Beacon cannot notify without app | QR/NFC, signage, merchant promotion |
| BB-18 | Fuel safety | No installation approved | OMC/site approval, hazard review, phone-use permission |
| BB-19 | Fuel payment/GHS source | Not approved here | Spendable GHS, provider contract, POS, reconciliation |
| BB-20 | Transit validation | No operator integration | Operator/validator/ticket API |
| BB-21 | Hotel digital key | No access provider | Provider, threat model, revocation/lockout |
| BB-22 | Commercial model | Not decided | Device ownership, support, replacement, merchant fee |
| BB-23 | Frequency policy | Not decided | User controls, global/per-vertical budget |
| BB-24 | Location claims | Must be conservative | Field evidence; no exact metres/position from RSSI |

## Non-negotiable boundaries

No inferred GHS funding or spendable balance from presence/USDC valuation; no treasury/fiat pool mutation; no payment dispatch/retry/result from detection; no static ID/client HMAC as physical proof; no raw scan counted as visit; no footfall claims; no battery/online claims without telemetry; no fuel install without written exact-location approval; no new personal-data use without notice/consent/governance.

## Decision log template

Record ID/date, question/options, owner/approver, evidence, chosen option/rationale, affected docs, implementation gate/rollback and scope of approval (prototype, pilot, production or commercial rollout).
