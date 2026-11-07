# 10 — Roadmap and acceptance criteria

## Stage 0 — Product, partner and compliance definition

Confirm owner-approved purpose/messaging; identify existing merchant/branch/permission records; obtain supplier RFI and model docs; verify NCA status; establish controller/processor roles, notice/consent with privacy review; select one or two retail/restaurant branches; define qualified session and retention.

**Exit:** prototype scope approved; compliance/privacy owner assigned; sample hardware justified; no fuel assumptions in generic prototype.

## Stage 1 — Hardware/native detection proof

Provision small sample; build registration/assignment; implement native Android/iOS detection; test foreground/background/locked/notification/force-quit; tune power/interval; measure phone battery/device reliability; simulate multiple stores and revocation.

**Exit:** defined supported matrix works; unsupported states honest; no mobile-embedded secret; unknown/revoked fails closed.

## Stage 2 — Presence API/templates/customer experience

Integrate existing merchant/branch/staff permission; add constrained versioned template; authenticated Flutter resolution; separate discovery/notification/personalisation; in-app card, notification and tap routing; multi-candidate choice; dedupe/session hysteresis/cooldown; truthful offline/permission fallback.

**Exit:** correct branch/template; arbitrary payload rejected; callbacks don't inflate visits; preferences enforced; no financial action from presence.

## Stage 3 — Portal and operations

Vite device/config page; tenant/branch scope; draft/publish/rollback; worker test/commissioning; Admin inventory/map; merchant-safe status/analytics; replacement/decommission/audit procedure.

**Exit:** merchant isolation; worker evidence separate; Admin reconciles assignments; health labels backed by source.

## Stage 4 — Retail/restaurant pilot

Limited branches, opt-ins, real-device Android/iOS matrix, adjacent-store false-positive observation, prompt/selection/dismissal/support/maintenance measurement, authoritative conversion linkage, privacy review.

**Exit:** pre-agreed reliability/usefulness thresholds met; false-positive acceptable; devices maintained; opt-outs work; no unresolved privacy/security incident.

## Stage 5 — Hotels/transit

Hotel requires property/reservation integration and guest privacy; room key is separate security project. Transit requires operator agreement, stop mapping, schedules and validator/ticket integration.

## Stage 6 — Fuel partner pilot (separate gate)

Only with willing OMC/station partner, exact site-safety approval, NCA/hazardous-area evidence, phone-use permission, approved GHS funding, provider contract, POS/attendant verification, dispense and reconciliation.

## Acceptance matrix

| Area | Must prove |
|---|---|
| Identity | Active ID resolves; unknown/revoked/expired fails |
| Assignment | Correct branch/zone; history auditable |
| Tenant security | Merchant cannot access other's devices/config |
| User auth | Server derives user; client-supplied ID cannot impersonate |
| Template | Approved schema; arbitrary script/URL/payment endpoint rejected |
| Foreground | Eligible session produces one card |
| Background | Notification only if OS event/permission available |
| Platform failure | Permission/Bluetooth denial yields honest fallback |
| Multiple stores | Not silently guessed from strongest signal |
| Session | Repeated callbacks do not create repeat visits |
| Cooldown | Dismissal/frequency limit suppresses |
| Personalisation | Opt-in; no automatic marketing notification |
| Offline | Stale event not current presence/transaction |
| Worker test | No customer visit/notification/transaction |
| Health | Claims only with supported telemetry |
| Analytics | Conversion from backend authority |
| Revocation | Revoked device stops resolving/audited |
| Payments | No payment/order/ticket/dispense from detection |
| UX | Understandable, dismissible, non-blocking |
| Operations | Install/move/test/replace/decommission documented |

## Definition of done

Controlled prototype requires Stage 0–2 and security/privacy tests. Field pilot adds Stage 3, partner consent and operations sign-off. Expansion requires evidence from prior stage. Fuel payment must pass its separate gates.

Green unit tests do not prove physical reliability, merchant readiness, lawful processing, provider contract, station safety or production readiness.
