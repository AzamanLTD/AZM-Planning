# Post-R15 Deep Repo Audit — 2026-09-20

Status: READ-ONLY audit findings after R15 merge. No implementation in this document.
Backend main audited: 754223eda926067580e6ff32027a00c49b3740ba
R15 PR: #287, merged to main.

## Release checkpoint

Main has two successful push runs on the merge SHA, both first attempt:
- Azaman Test Suite: 35522036669 — SUCCESS.
- financial-durability: 35522036670 — SUCCESS.
The final PR evidence reports 251/251 suites and 2,018/2,018 tests; financial durability 31/31 suites and 306/306 tests; route-check PASS; recovery drill SUCCESS; Prisma clean; production dependency audit 0 vulnerabilities.
This proves R15, not every older financial/product domain.

## P0 — Moolre-only product contract is not yet reflected in provider topology

Product decision: Moolre is the only current external fiat provider contract. Current code still instantiates Moolre as primary and a direct MtnDisbursementService as secondary in PaymentFailoverService. Provider ownership also maps an mtn failover tag to MTN_MOMO_DISBURSEMENT.
This means a safe failover can still place customer money onto a direct MTN provider path, which conflicts with the current product architecture.
Required direction: canonicalize local fiat behind Moolre. Network choice is destination information under Moolre. Do not retain a direct MTN rail without a separately approved real contract.

## P0 — admin fiat rejection uses the wrong obligation reference

finance.service creates the P4/P5-D restricted obligation as withdrawal:fiat:<transaction-history-reference>. adminController.rejectWithdrawal searches only withdrawal:wallet:<withdrawal-id> and withdrawal:smartroute:<withdrawal-id>.
Observed consequence: a normal P5-D fiat withdrawal can be rejected while its active restricted obligation is not found; the user balance is refunded but the ledger can fall back to equity and the real restricted obligation is not cancelled.
The fix must use the canonical finance reservation identity rather than add another special case.

## P0 — Smart Route remains execution-unsafe and outside R15

Current SmartRoute execution remains read -> decision -> action -> schedule advancement. There is no durable per-occurrence claim. The manual run-now endpoint calls the same runOnce path independently of the scheduled worker, so manual and scheduled execution can overlap.
MoMo creates a pending Withdrawal and a SMART_ROUTE_RUN history row marked COMPLETED before provider settlement; dispatch is separate and errors are swallowed. It is not aligned with the canonical finance withdrawal identity.
Internal transfer decreases availableBalance, credits the recipient, and also increments the sender escrowLockedBalance even though no escrow exists.
Savings execution reads the goal outside the transaction, then writes it inside without a single-winner occurrence identity.
Required direction: evolve Smart Route behind the generalized scheduled payment/payout architecture instead of adding more action-specific patches.

## P0 — Payroll and EWA tests prove the wrong money semantics

Product contract: azmBalance is AZM loyalty points; payroll/EWA are monetary USDC obligations.
Current PayrollService.disbursePayroll increments user azmBalance. Current EwaService.requestWithdrawal also increments user azmBalance.
The integrity suites explicitly assert those writes, so green tests currently protect behavior that conflicts with the canonical product financial model.
EWA also hardcodes a 1 percent fee in backend and Flutter UI.
Required direction: move payroll/EWA onto monetary liability and business source-of-funds accounting, then rewrite the tests to prove that contract.

## P0 — Savings withdrawal has no concurrency proof and a deeper denomination problem

Withdrawal reads SavingsGoal.currentAmountGhs outside the transaction, computes the FX conversion outside the transaction, and updates the goal from that stale snapshot. The transaction reference is generated from the current timestamp and is not caller-idempotent.
The existing test only proves a sequential second full withdrawal fails after the first makes the goal CANCELLED; it does not prove concurrent partial withdrawals are safe.
There is also a denomination issue: the goal is GHS-denominated while the locked customer liability is USDC. Releasing GHS at the current FX rate can differ from the exact USDC amount that funded the saved value.
Required direction: define exact savings denomination semantics first, then add a DB-backed single-winner withdrawal claim and stable idempotency.

## P0 — Order Book has a direct runtime bug and weak concurrency protection

In controllers/orderBookController.js the BUY path references order.id in the ledger idempotency key before const order is created. This is a JavaScript temporal-dead-zone runtime error.
An orderBookRoutes.js route is mounted, but no dedicated order-book integrity suite was found in the repository test tree.
The matching engine reads resting orders into a stale candidate list and then updates remaining quantities unconditionally, allowing concurrent takers to race for the same resting liquidity.
There is also an unresolved product decision: the book trades AZM/USDC while azmBalance is defined as loyalty points. The current settlement path has no separate authoritative AZM asset ledger.
Do not silently turn loyalty points into a tradable monetary asset; decide whether this feature is current product scope or legacy.

## P1/P0 exposure — Shared Vault is a stale broken financial route

sharedVaultRoutes.js is mounted. sharedVaultController.js creates its own PrismaClient and calls prisma.wallet, while the current schema does not contain a Wallet model.
The route also uses its own wallet/vault accounting instead of the canonical liability ledger.
Required direction: formally retire/disable it or rebuild it against the canonical ledger. Do not invent a Wallet model merely to revive the old implementation.

## P1 — identity and client contracts have not converged

Peer transfer remains friendship-ID based rather than first-class AZM-ID payment targeting. Business Portal employee creation sends azmId while EmployeeService.addEmployee expects numeric userId and does not resolve AZM-ID.
Flutter worker EWA and Business Portal payroll surfaces exist, but the backend semantic layer is still inconsistent with the intended monetary model.

## Bottom line

R15 is successfully merged and green on main. The remaining risk is no longer the R15 payout-reconciliation layer itself; it is older financial domains whose code, contracts and tests have not yet converged on the new architecture.
The highest-leverage next implementation boundary is to close the existing financial authority mismatches before building the generalized scheduled-payment engine:
1. admin fiat rejection reference and provider-topology cleanup;
2. payroll/EWA monetary accounting and configurable policy;
3. then Smart Route redesign into generalized PaymentPlan execution.

Green CI must always be interpreted as evidence for the exact contract the tests assert, not as proof that the product architecture is correct.