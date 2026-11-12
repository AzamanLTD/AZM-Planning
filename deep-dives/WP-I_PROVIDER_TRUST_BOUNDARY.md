# WP-I — Provider trust boundary and treasury evidence

**Status:** investigation complete, implementation NOT started. Release blocker for treasury funding. Part II (§7-§14) corrects three provider-contract overstatements in Part I and adds funding-type applicability, gross/net/currency semantics, the credential model and callback verification for the real Moolre contract.
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

**Corrected (Part II §7):** the provider *recommends* verifying state with a status endpoint for high-value or unexpected payloads. It does not prescribe, mandate or describe a separate reconciliation process, a second credential or an independent database identity. The separate-process treasury architecture is **our security requirement**, built on the verification capability the provider does expose.

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

---

# PART II — Provider-method applicability, gross/net and process isolation

Added after reviewing the official Moolre documentation set in full. Part II **supersedes** the corresponding provider claims in §3.3 where they differ.

## 7. Corrections to Part I

| Part I claim | Correction |
|---|---|
| "The provider's own documentation prescribes the architecture recommended in §4." | The provider *recommends* status-endpoint verification for high-value or unexpected payloads. It does not prescribe a separate process, a second credential or an independent database identity. That architecture is **our** requirement, enabled by a capability the provider exposes. |
| `X-API-PUBKEY` treated as though it were a secret | Moolre describes it as the **Public API Key**. It is not a hidden credential and must not be presented as one. Genuinely private material is `X-API-KEY` (Private API Key), used by the statement and account-status APIs only. |
| One "Payment Status API" treated as the witness for treasury inflows | `/open/transact/status` with `type=1` proves **only a collection Azaman itself initiated and holds the externalref for**. It cannot prove a bank deposit, an unsolicited mobile-money transfer or a wallet-to-wallet credit. See §8. |

## 8. Funding-type applicability — what each witness can actually prove

`FiatLiquidityReceipt` stores `provider`, `providerRef` (nullable), `dedupKey` (economic identity), `amountGhs` (2dp) and `status`. It stores **no account number, no external reference and no currency** (`prisma/schema.prisma`, model `FiatLiquidityReceipt`). Whatever the reconciler queries, it must first be given a server-created identity and a stored account mapping.

| Funding method | Can `/open/transact/status` type=1 prove it? | Correct independent witness | Credential used |
|---|---|---|---|
| Collection initiated by Azaman through Moolre, with an Azaman `externalref` | **Yes** — `idtype=1` by our externalref, or `idtype=2` by Moolre transaction id. Proves *only* that collection; not unsolicited or unrelated credits | Payment Status API (`/open/transact/status`, `type=1`) | `X-API-PUBKEY` (public) |
| Bank transfer into a Moolre virtual account | **No** — `type=1` keys on a collection `externalref` we generated; a deposit carries the **account's creation reference**, not a collection reference. Public key is sufficient for the correct witness | List Bank Account Transactions — `/open/transact/status` with `type=2`, keyed by the account's **creation** externalref + `accountnumber`, returning a transaction array | `X-API-PUBKEY` (public) |
| Mobile-money transfer into the wallet, not initiated through our collection API | **No** | List Transactions (wallet statement) — `/open/account/status` with `type=2`, `accountnumber`, date range, optional status filter; match on `transactionid` / `thirdpartyref` / `amount` / `value` / `ts` | **`X-API-KEY` (private)** |
| Wallet-to-wallet (internal transfer) received | **No** — internal transfer is outbound from the sending wallet; the credit appears on the receiving wallet's statement | List Transactions (wallet statement) | **`X-API-KEY` (private)** |
| Settlement / disbursement | Not an inflow | Transfer Status API (outbound lifecycle only) | `X-API-PUBKEY` (public) |
| Any other credit to the wallet | **No** | Wallet statement, matched conservatively | **`X-API-KEY` (private)** |

**Consequences for the current model.** `confirmTreasuryOpening` accepts *any* INBOUND event with a settled status string, provider `MOOLRE`/`MTN_MOMO`/`AZM_WEBHOOK` and a matching amount (`src/services/fiatLiquidityService.js:127-128`, `:1028+`). It does not distinguish a collection we initiated from a bank deposit — a distinction the provider's APIs make absolutely. The treasury-opening path therefore needs an explicit **funding type** on the receipt before any witness can be selected, and the witness differs per type.

**What each witness can and cannot reach.** These are three distinct capabilities, and conflating them is how a wrong key gets chosen for a lookup:

- **`type=1` (collection):** proves the outcome of a mobile-money collection *Azaman itself initiated and can name*. The lookup key is our own `externalref` (`idtype=1`) or the Moolre transaction id (`idtype=2`). It says nothing about unsolicited credits, bank deposits, or any transaction we did not start, and must never be presented as a general proof of wallet inflow.
- **`type=2` (bank virtual account):** reaches credits into a bank virtual account using that account's **creation** external reference plus its `accountnumber`, returning a transaction array that must then be matched to the opening. This is an ordinary **public-key** call; a virtual-account deposit does **not** require the private wallet-statement path.
- **Wallet statement (`/open/account/status` `type=2`):** the only route to credits that have no pre-existing collection or virtual-account identity — unsolicited mobile-money credits, wallet-to-wallet receipts, anything else landing in the wallet. This is the **private-key** path.

**Where the public API is genuinely insufficient.** Only the third category requires the private key. If Azaman will not place the private key in the reconciler, that category stays **unavailable** for treasury funding until a provider statement, reconciliation export or written confirmation is obtained. Bank virtual-account deposits remain available through `type=2` with the public key, subject to the matching rules above.

## 9. Gross amount, net credited value and currency

The documented webhook payload carries both `amount` and `value`, defining `value` as "Transaction value after applicable fees" (`docs.moolre.com/api/payments/webhook`, field table). The statement APIs return the same pair per transaction.

**Defect in current code (exact candidate):** `controllers/moolreQuoteDepositController.js:361` parses `data.amount` — the payer's gross figure — as the settled amount. Where a fee is deducted from the credit, our evidence is overstated by that fee, and `confirmTreasuryOpening`'s exact-equality check then compares gross-to-gross, making the overstatement internally consistent and therefore invisible.

**Rule for treasury evidence:** reconcile to the **net credited value that the wallet actually received**, proven from the provider response for that specific transaction — never to the payer's gross, and never to an amount our own normaliser asserted. Gross and net legitimately differ; which side bears the fee appears to depend on pricing band ("Special payment pricing" sections), which the public documentation does not resolve into a single rule. Until it is resolved, a path whose gross-vs-net relationship is unproven stays **unavailable**.

**Currency:** neither the callback nor the status/statement responses carry a currency field; the webhook exposes `accountnumber` (destination wallet) and the statement exposes `balance` unlabelled. Currency is established at **account creation** (Bank Collections API `type=9` takes `currency: GHS | NGN`). Therefore:
- the account-number → currency mapping must be a **stored, controlled table** owned by the treasury side, populated from account creation records, not inferred per transaction;
- the mapping is **owned by the treasury side and must be protected from the main runtime credential** — a compromised application must not be able to add an account or relabel its currency;
- each mapping entry must be **grounded in the virtual-account creation record** (the `type=9` creation response), not in a value asserted at callback time;
- the reconciler must refuse to emit a GHS witness for an account absent from that mapping;
- `FiatLiquidityReceipt.amountGhs` presupposes GHS; nothing in the provider response independently proves it.

## 10. Credential and process boundary (corrected)

| Credential | Nature per provider docs | Where it may live |
|---|---|---|
| `X-API-USER` | account identifier | reconciler and app |
| `X-API-PUBKEY` | **Public** API Key — not a secret | either |
| `X-API-KEY` | **Private** API Key — genuinely secret | **reconciler only** |

The key hierarchy is narrower than a first reading suggests, and it is **not** the security boundary:

- Moolre **names `X-API-PUBKEY` the Public API Key**. Withholding a public key protects nothing. The collection lookup and the bank virtual-account lookup are both public-key calls, and neither is made trusted by keeping that key away from the application.
- `X-API-KEY` is the **Private API Key**. It is required for the general wallet statement and account-status endpoints — the path that reaches arbitrary wallet credits. That is the one credential the main API process must not hold, and only *where that path is in use*.
- Observable ledger state (collections, bank virtual-account credits) may therefore be reconciled by a component holding nothing secret. **A separate reconciliation writer and evidence authority is still required in that case**, because the boundary being defended is *who may author evidence*, not *who holds a key*.

The separate reconciliation service is required for reasons that survive the public-key observation:

- it is the only component permitted to write reconciliation witnesses;
- it holds the private key only when the statement path is in scope, and never in the API process;
- it cannot mutate treasury openings or economic balances, and the runtime cannot write a witness or promote a receipt to `AVAILABLE`;
- it fetches evidence from Moolre itself and binds it to an opening the server already created.

**Bank-account `type=2` being a public-key path does not weaken any of this.** It lowers the credential cost of that one funding type; it does not make application-authored rows trustworthy, and it does not permit the application to emit or promote a witness.

Required properties of the reconciliation boundary:

1. A separate process/service with its own deployment and egress identity, not a request handler inside the API.
2. It holds the private key **only where the statement path is in scope**; the API process never holds it. For collection and bank virtual-account lookups no secret is involved, and the boundary still applies.
3. It connects to PostgreSQL under its **own** database identity, with `INSERT` only on the witness table and `SELECT` only on the pre-existing treasury openings it must match.
4. It queries the provider over HTTPS itself — never in response to a user-supplied reference or amount.
5. The lookup key is a **pre-existing server-created treasury opening** (its dedupKey plus stored account and reference), not anything the application or a user supplies at query time.
6. Matching is exact on provider account, external reference, net amount, currency (via the mapping) and settled state.
7. It stores the **immutable provider response**: raw body, retrieval timestamp, response digest, endpoint and request parameters.
8. No application-supplied claim may select, parameterise or manufacture the provider result.
9. Receipt binding stays single-use, with contradiction quarantine for byte-different but economically identical observations.
10. Its **database identity cannot mutate treasury openings or economic balances** — it may not change `FiatLiquidityReceipt.status`, `fiatLiquidityState`, ledger rows or `availableGhs`, and only the promotion path (not the runtime) may move a receipt to `AVAILABLE`.
11. It **rejects any instruction carrying an amount or reference** supplied by the application or a user; the only accepted input is the identifier of an opening that already exists.

**How a compromised application process is kept out:** it can run *any SQL the runtime role may run* and call *any endpoint the app exposes*, but it cannot (a) write or alter a witness, (b) instruct the reconciler to look up an identifier it chose, or (c) fabricate the provider's HTTPS response and its digest — and, where the statement path is in scope, (d) read the private key. Note that (a)–(c) hold even for the public-key funding types: the defence is the absence of write and instruction authority, not the secrecy of a key. The reconciler must therefore accept **no inbound instruction carrying an amount or reference**; its only trigger is "reconcile opening X", where X must already exist in the database with a stored account and reference. Anything else reintroduces the forgery primitive this document exists to remove.

## 11. Callback verification for the actual Moolre contract

1. **Secret location:** `data.secret` inside the JSON body; compare against our stored expectation in constant time. Never echo it; redact it from retained raw payload.
2. **Source-IP allowlisting** at the network edge (proxy/LB), using the published callback IP list, as an additional control — not the primary one.
3. **Proxy and client-IP handling:** trust only the platform's own forwarding header, from known proxy addresses; never `X-Forwarded-For` as supplied by the caller. Reject when the effective client IP cannot be established.
4. **Raw payload retention and digest:** store the exact bytes received plus a SHA-256 digest and receipt timestamp, before any parsing-derived state is committed.
5. **External-reference correlation:** correlate on `data.externalref` for collections we initiated; a callback whose reference is unknown must be recorded and **not** bound to any receipt.
6. **Exact decimal and account validation:** parse `amount`/`value` through the existing exact-decimal authority; validate `data.accountnumber` against the stored account→currency mapping; treat `status=0` / `P02` / `txstatus=2` as failure and `txstatus=0` as pending, never as settled.
7. **Duplicates and contradictions:** idempotent on the economic identity; byte-different but economically identical observations converge; materially different observations are quarantined, never overwritten.
8. **Mandatory independent reconciliation** before an observation becomes treasury-eligible (Part II §10). The callback alone — body secret, IP allowlist, or both — never makes a receipt `AVAILABLE`.
9. **Remove** the undocumented static-header fallback at `controllers/moolreQuoteDepositController.js:344` and the static-first ordering at `controllers/fiatSettlementWebhook.controller.js:35-40`. Do **not** replace them with an invented HMAC; if Azaman wants HMAC, the exact algorithm, encoding and signed bytes must be confirmed in writing by Moolre first.

## 12. Additional acceptance cases (matrix now 26)

Each negative case must prove: no receipt reaches `AVAILABLE`, `FiatLiquidityState.availableGhs` is unchanged, and no duplicate economic effect occurs.

| # | Case | Required outcome |
|---|---|---|
| 19 | Funding type not supported by the selected status endpoint (bank deposit queried as `type=1`) | Witness refused as inapplicable; receipt stays `RECEIVED` |
| 20 | Status response cannot prove GHS (account not in currency mapping) | Refused; stays `RECEIVED` |
| 21 | Gross `amount` vs net `value` mismatch on a fee-bearing payment | Refused or reconciled to net only; never credited gross |
| 22 | Provider returns pending / non-final (`txstatus=0`, or no terminal state) | Stays `RECEIVED`; no promotion |
| 23 | Reconciliation service unavailable | Fail closed; `RECEIVED`; alert |
| 24 | Application submits an arbitrary reference with no pre-existing treasury opening | Reconciler refuses to query; no witness written |
| 25 | Reconciler process compromise (simulated: witness written without a provider call) | Detected by digest/timestamp/endpoint binding; receipt not promoted |
| 26 | One provider observation matched to a second treasury opening | Refused by single-use binding; second receipt stays `RECEIVED` |

## 13. Remaining provider questions (supersedes §3.5 items)

1. Written confirmation of the authoritative callback authentication method, including whether any HMAC exists and its exact specification.
2. Whether bank credits to a virtual account fire a callback at all — no documented callback was found.
3. `txtype` semantics (examples show `1` and `2`, no definition found); whether `1` reliably denotes a credit.
4. Which party bears fees per pricing band, and therefore when `value < amount`.
5. Statement and transaction-list coverage as an audit witness: pagination, `limit` semantics, maximum date range, retention horizon, and whether a completed list for a window is provably complete rather than merely what was returned.
6. Whether Moolre provides settlement statements, reconciliation exports or a written statement of account suitable for audit.
7. Whether the published callback IP list is stable, and the expected behaviour behind a CDN or proxy.
8. The **account-level permission scope** of each credential: the documentation names `X-API-PUBKEY` (Public) and `X-API-KEY` (Private) but does not publish a complete endpoint-by-endpoint permission matrix, nor whether either key can be issued with narrower scopes.

## 14. Verdict (updated)

Treasury funding remains **NOT READY**. Two defects stand: (a) a runtime database credential can fabricate the evidence `confirmTreasuryOpening` consumes — proven on `6a6f233`; (b) the evidence amount is the payer's gross figure rather than the net credited value, so even honest evidence can overstate the treasury by the fee. Neither the callback body secret nor IP allowlisting closes (a). For (a), only evidence fetched independently by a component the application can neither instruct nor impersonate closes it; the private key is required only for the statement category, while collection and bank deposits need no secret at all and still require the separate writer.
