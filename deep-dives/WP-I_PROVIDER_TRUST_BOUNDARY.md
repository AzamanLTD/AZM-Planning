# WP-I — Provider trust boundary and treasury evidence

**Status:** investigation complete, implementation NOT started. Release blocker for treasury funding.
**Author role:** independent review (read-only). No backend source was modified.
**Reviewed candidate:** `AzamanLTD/AZM-backend` PR #399, head `6a6f23304c7a2fdfe8e3e0a60b61dda97e07ebd3`, base `main` `0edefa1dbe`.
**CI on that exact SHA:** Azaman Test Suite `43116890057` success (445 suites / 4683 tests, first attempt, no retry); financial-durability `43116890047` success (62 suites / 702 tests).
**GitHub record:** formal blocking review on PR #399 (review id `5867229526`). GitHub refuses `REQUEST_CHANGES` on a PR authored by the same identity, so the blocking review was submitted with event `COMMENTED` and titled REQUEST CHANGES.

---

## 1. Finding

A restricted runtime database login can manufacture treasury funding evidence, and the confirmation path accepts it.

`confirmTreasuryOpening` requires a `FiatProviderEvent` identity, then checks: provider in `['MOOLRE','MTN_MOMO','AZM_WEBHOOK']`, `direction === 'INBOUND'`, `status` in `['FUNDED','SETTLED','SUCCESS','PAID','COMPLETED']`, exact `amountGhs` equality with the receipt, and single-use binding. Every one of those is a column value the application's own credential can author, and `FiatProviderEvent` carries no provenance field.

The change that introduced this gate (requiring a settled inbound observation instead of a caller-supplied funding reference) is a genuine improvement. It removes caller assertion. It does not remove application authorship.

### Evidence classes

| Class | Content |
| --- | --- |
| **Executed on exact candidate** | Runtime-role event insertion; `confirmTreasuryOpening` acceptance; control refusal; callback authentication probes |
| **Source-inspected on exact candidate** | Route mounts, auth branches, writer inventory, gate predicates, deployment configuration |
| **Provider documentation** | Moolre official docs, cited in §3 |
| **Unverified operational assumptions** | Live Render environment, provider dashboard configuration, real callback payload shape observed in production |

---

## 2. Exact-candidate source map

All paths below are on `6a6f233`.

### 2.1 Callback routes

| Route | Mount | Authentication |
| --- | --- | --- |
| `POST /api/deposit/fiat/webhook` | `routes/depositRoutes.js:32` | `x-azaman-webhook-secret` compared to `FIAT_WEBHOOK_SECRET` (`controllers/quoteFiatDepositController.js:237-239`) — shared secret, **no body binding** |
| `POST /api/deposit/fiat/webhook/moolre` | `routes/depositRoutes.js:39` | HMAC-SHA256 over `req.rawBody`, hex, timing-safe (`controllers/moolreQuoteDepositController.js:340-343`), then **static-header fallback** at `:344` |
| `POST /api/finance/webhook/moolre-disbursement` | `routes/financeRoutes.js:83` | static secret checked **first** (`controllers/fiatSettlementWebhook.controller.js:35-36`); signature headers only afterwards (`:40`) |
| MTN branch on that route | same | static `x-mtn-webhook-secret` equality only (`:55-59`), no HMAC |

Raw-body capture is correct: `src/middleware/appMiddleware.js:99` stores `buf.toString('utf8')` on `req.rawBody`, so the HMAC covers the exact wire bytes when the header is present.

No MTN MoMo inbound route exists in `routes/`. MTN observations enter through service/recovery paths only. This document does not propose adding one.

### 2.2 Event writers

Direct Prisma writes, all inside `src/services/fiatLiquidityService.js`: `:271` `updateMany` (claim), `:525` `create`, `:568` `create`, `:1108` `update`.

`recordProviderEvent` is exported (`:1830`) and has 15 call sites across 11 files. It validates shape only (direction, required fields, amount on INBOUND). It records no provenance.

### 2.3 Writer classification

Blanket write revocation would break payouts. Each writer is classified by what it is entitled to prove:

| Writer | Class | May prove treasury funding? |
| --- | --- | --- |
| `controllers/moolreQuoteDepositController.js` (`:219`, `:439`) | Inbound provider observation (collection) | Yes, but only via verified callback |
| `controllers/quoteFiatDepositController.js` (`:289`) | Inbound provider observation (generic) | Yes, but only via verified callback |
| `controllers/fiatSettlementWebhook.controller.js` (`:130`) | Outbound disbursement callback | **No** |
| `controllers/withdrawalController.js` (`:514`, `:671`) | Outbound disbursement | **No** |
| `services/moolreCollectionRecoveryService.js` (`:182`, `:221`) | Recovery / polling | Only with independent provider status evidence |
| `services/nativeGhsWithdrawalService.js` (`:1558`, `:1665`) | Outbound disbursement | **No** |
| `services/fiatSettlementService.js` (`:85`) | Settlement bookkeeping | **No** |
| `services/smartRouteService.js` (`:1117`, `:1245`) | Internal routing/bookkeeping | **No** |
| `workers/payoutBatchWorker.js` (`:635`) | Internal worker bookkeeping | **No** |
| `workers/withdrawalReconciliationWorker.js` | Reconciliation | **No** |
| `src/services/fiatLiquidityService.js` internal `create`/`update` | Internal state machine | **No** |

The design consequence: treasury eligibility must be a **separate property**, not a restriction on the table. Outbound and internal writers keep working; only inbound evidence that satisfies §4 may back a treasury opening.

---

## 3. Provider contract evidence

The implementation is evidence of what our code expects, not proof of what the provider sends. The current HMAC is **not** the documented Moolre contract.

### 3.1 Moolre collection callbacks

- Source: <https://docs.moolre.com/api/payments/webhook> (Payment Webhook API). Failed-callback and source-IP documentation added per changelog entry dated **7 Oct 2026**, <https://docs.moolre.com/changelog>.
- Documented verification: a `secret` field **inside the callback body** — "Callback verification secret. Validate it against the expected secret for your integration" — plus callback source-IP allowlisting ("Allowlist all the listed addresses"), described as "an additional safeguard, not a replacement for callback verification".
- Documented payload: `status` (1 success / 0 failure), `code` (`P01` / `P02`), `data.txstatus` (1 successful, 0 pending, 2 failed), `data.amount` (decimal **string**), `data.value`, `data.transactionid` (nullable), `data.externalref`, `data.ts`.
- **No HMAC header, algorithm or signed-bytes definition is documented.** The `x-moolre-signature` HMAC in our code is not documented by the provider.
- Explicit provider statement: "A callback arriving at your server is not itself proof of a successful payment."

Consequence: the documented contract is a **shared secret transported in every callback** plus IP allowlisting. That is not payload-bound either. Neither our HMAC nor our static fallback matches the documentation, and the documented mechanism does not by itself establish authenticity to the standard treasury funding requires.

### 3.2 Moolre disbursement callbacks

The documentation set describes transfer lifecycle through the **Transfer Status API** (<https://docs.moolre.com/api/transfers/status>) and the transfers guide (<https://docs.moolre.com/guides/transfers>). No signed disbursement callback contract was found. Outbound bookkeeping therefore cannot be treated as inbound evidence, which matches the classification in §2.3.

### 3.3 Moolre settlement and reconciliation

- **Payment Status API** — `POST https://api.moolre.com/open/transact/status`, headers `X-API-USER` / `X-API-PUBKEY`, body `type=1`, `idtype=1` (unique externalref) or `2` (Moolre ID), `id`, `accountnumber`. This is an **independently fetchable provider-side witness** for a collection.
- **List Payment ID Transactions** (<https://docs.moolre.com/api/payments/payment-id/transactions>), **Accounts transactions** (<https://docs.moolre.com/api/accounts/transactions>), **Account status/settlement** (<https://docs.moolre.com/api/accounts/update>, `/api/accounts/status`).
- Provider guidance, "Verify the final state" (<https://docs.moolre.com/webhooks>): "For high-value actions or an unexpected payload, verify the state with the relevant status endpoint before completing the customer-facing workflow." Also "Reconcile uncertain outcomes": keep the operation pending on timeout, confirm via status or callback.

**The provider's own documentation prescribes the architecture recommended in §4.** That is the strongest available contract evidence for this design.

### 3.4 MTN MoMo

No official MTN inbound-callback contract is recorded for this integration in the reviewed candidate, and no inbound route exists. MTN appears only as a provider label and through recovery paths. This document does not assume MTN sends callbacks.

### 3.5 Provider-contract questions the owner must close

1. Written confirmation from Moolre of the authoritative callback authentication method: body `secret` only, or an HMAC we have misread. Include the exact header name, algorithm, encoding and signed bytes.
2. Whether `x-moolre-signature` and `x-moolre-webhook-secret` are provider-defined or integration-invented headers.
3. Whether Payment Status (`/open/transact/status`) is contractually stable for reconciliation and what its rate limits and retention are.
4. Whether a dated settlement statement or report export exists for daily reconciliation.
5. Whether IP allowlisting addresses are fixed and published for our environment, and what the provider expects behind a proxy/CDN.

---

## 4. Recommended evidence model

### 4.1 Threat model, by adversary

| Adversary | Current candidate | Required target |
| --- | --- | --- |
| Forged external callback, no secret | Blocked (401) | Blocked |
| Forged callback with the shared secret, or a captured one | **Accepted** (static paths; body `secret` per provider docs is also transferable) | Acceptance **limited**: callback alone never makes a receipt `AVAILABLE` |
| **Compromised runtime database credential** | **Succeeds — executed proof** | **Blocked**: eligibility requires a provider witness the credential cannot mint |
| **Compromised main application process** | **Succeeds** (holds runtime URL, provider keys, in-process writers) | **Blocked** only by an independently fetched provider witness with its own credentials and its own egress identity |
| Compromised provider-ingestion process | Succeeds | Contained: it can write observations but cannot alone make them authoritative |
| Captured / replayed genuine callback | Constrained (`dedupKey` unique, single-use binding) | Unchanged, plus idempotent provider-status confirmation |
| Privileged operator misuse | Unattributable | Attributed, append-only activation record |
| Provider timeout then delayed settlement | Safe by construction (nothing becomes `AVAILABLE` without settled evidence) | Unchanged; reconciliation converges on one economic identity |

A second credential inside the same Node process is **not** a process-compromise defence. This document does not claim otherwise.

### 4.2 Layered recommendation

**Authoritative layer — independently fetched provider settlement witness (option 4/5).** A treasury receipt may become `AVAILABLE` only when a reconciliation record matches, on all of: provider, external reference, currency GHS, exact amount in pesewas, and a settled state reported **by the provider** at fetch time. The fetch uses a dedicated reconciliation identity with its own credential and egress, and the matched result is stored immutably with its fetch timestamp and a digest of the provider response. This is the only layer that survives compromise of the main application process, and it is what Moolre's own documentation recommends.

**Payload-bound signed observation (option 1) — only if the provider contract supports it.** Moolre documents a body `secret` and IP allowlisting, not a signature. Until §3.5 item 1 is answered in writing, do not build on an assumed HMAC. If a real signature is confirmed, store it with the raw body digest and **re-verify at confirmation time**, not only at ingestion.

**Restricted ingestion identity (option 2).** Useful as defence in depth against a stolen runtime credential, provided ingestion is a **separate process** with its own credential. Inside the same process it is theatre.

**Database-enforced provenance (option 3).** A `SECURITY DEFINER` ingest function owned by a non-login role, runtime `INSERT` revoked on `FiatProviderEvent`, plus a provenance column with a `CHECK` that treasury eligibility can only be set by that function. This closes credential compromise cleanly and prevents accidental regressions; it does not close process compromise.

**Single-use binding and replay (existing).** Keep `dedupKey` uniqueness, `relatedReference` binding and the CAS claim. They are sound and already tested.

### 4.3 Residual risk statement

With all four layers, a fully compromised application **process** still cannot mint treasury funding **provided** the witness fetch is performed by a separate component holding a credential the app never sees. If the owner declines a separate reconciliation component, the honest statement is: credential compromise is closed, process compromise is not, and the compensating control must be human review of every treasury opening against a provider statement before activation.

---

## 5. Test matrix (acceptance)

Every negative case must assert: no receipt reaches `AVAILABLE`, `FiatLiquidityState.availableGhs` is unchanged, and no second economic effect occurs.

| # | Case | Expected |
| --- | --- | --- |
| 1 | Inbound event inserted by runtime role | Refused as evidence |
| 2 | Invalid HMAC | 401, no row |
| 3 | Missing HMAC (no static header) | 401, no row |
| 4 | Static-secret-only header, mismatched body | 401 |
| 5 | Valid signature, altered amount | Refused / contradictory evidence |
| 6 | Valid signature, altered reference | Refused |
| 7 | Replay of identical callback | One economic effect |
| 8 | Byte-different, economically identical | Converges |
| 9 | Contradictory repeated observation | Retained as distinct, never absorbed; typed error |
| 10 | Wrong provider | Refused |
| 11 | Wrong direction | Refused |
| 12 | Wrong amount (off by one pesewa) | Refused |
| 13 | Wrong currency | Refused |
| 14 | Unsettled status | Refused |
| 15 | Already-bound observation | Refused |
| 16 | One observation funding two openings | Second refused |
| 17 | Timeout, then settlement, then reconciliation | One receipt, one `AVAILABLE` transition |
| 18 | Positive control with genuine provider evidence | Confirms exactly once |

When implemented, the suite **must** be appended to the explicit `SUITES` array in `.github/workflows/financial-durability.yml`. That lane runs an array, not a Jest pattern; an unlisted file is silently never executed, and a green lane proves nothing about it.

---

## 6. Open owner decisions

1. **Witness authority.** Approve an independently fetched provider settlement witness as the authority for treasury funding. Without it, WP-I cannot be closed to process compromise.
2. **Ingestion topology.** Separate ingestion/reconciliation component (stronger) versus additional credential in the same process (weaker, must be documented as such).
3. **Callback authentication.** Adopt the documented body-`secret` + IP allowlist, or obtain a written provider signature contract and implement that. Remove the undocumented static header fallback either way.
4. **MTN scope.** Confirm whether MTN is expected to send inbound callbacks at all; today no route exists and none is proposed.
5. **Activation posture.** Treasury funding stays closed until the witness layer and the test matrix are in place.

---

## 7. Read-only attestation

No backend implementation, no provider calls, no credential or secret changes, no role provisioning, no Render changes, no Redis changes, no activation, merge or deployment. The executed evidence was produced on a disposable local PostgreSQL database with a synthetic least-privilege role; no production or shared environment was touched, and no connection details are recorded here.
