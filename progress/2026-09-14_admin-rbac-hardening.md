# 2026-09-14 Admin RBAC / control-plane hardening — verified remediation

## Current-main finding and remediation

The Admin RBAC controller previously defined a granular permission catalog but route-level authorization only enforced that the caller was some recognized admin. Approval creation, approval, rejection, audit export, approval listing, and Susu health therefore did not consistently enforce the intended permission matrix.

The controller also eagerly instantiated a private `PrismaClient`, creating a second database authority instead of using the application-scoped Prisma instance.

The approval state machine had two correctness issues: non-monetary actions such as USER_BAN/VENDOR_TIER_CHANGE could fall into the zero-approval monetary tier, and approval/rejection mutations used read-then-write behavior that could lose concurrent approvals or race terminal state transitions. Audit CSV export also duplicated `targetType` into the `targetId` position, shifting subsequent columns.

## Verified implementation

`AzamanLTD/AZM-backend#240` implemented the hardening in the canonical controller:

- removed the eager `PrismaClient` and switched handlers to `req.app.get('prisma')`;
- enforced action-specific approval permissions at creation, approval, and rejection;
- kept VENDOR_TIER_CHANGE restricted to SUPER_ADMIN / legacy ADMIN until its permission catalog is deliberately extended;
- made USER_BAN and VENDOR_TIER_CHANGE require one additional approval rather than auto-approval;
- enforced monetary-tier `requiredRoles` and the >= $50,000 Finance/Compliance participation invariant;
- made approval writes compare-and-swap safe using `updateMany` guarded by request status and JSON `approvals` equality;
- returned deterministic 409 conflict semantics for expected approval/rejection races instead of raw Prisma errors;
- made rejection transitions atomic against the PENDING state;
- enforced `audit.view`, `audit.export`, and `susu.health` for their respective read endpoints;
- corrected audit CSV field alignment to eight headers/eight fields.

## Regression coverage

`__tests__/admin-rbac-approvals.test.js` adds mocked policy/state-machine coverage plus CI-gated live Postgres concurrency coverage. The exact-head CI run verified:

- permission enforcement and denial cases;
- non-monetary auto-approval prevention;
- sub-$1k monetary auto-approval preservation;
- self-approval/duplicate prevention;
- monetary role-tier enforcement;
- >= $50,000 Finance/Compliance participation;
- compare-and-swap approval conflicts;
- concurrent approve/reject terminal-state integrity;
- concurrent reject integrity;
- audit CSV column alignment;
- real Postgres approval concurrency and JSON-equality CAS behavior.

## Verification and merge

- Backend PR: `#240`
- Exact verified PR head: `477305f9e705bf3f2f33b47a53e2f158b5380cb7`
- Canonical CI run: `#922`
- All canonical CI stages passed: dependency install, production dependency audit, schema application, Prisma generation, full Jest suite, route registry verification, database backup/restore drill.
- Merged to backend main as: `d745bbbd3de9353df4796aaa843aafbb264a1a28`

## Explicit policy tension retained for follow-up

The current Compliance Admin role includes `withdrawals.review` and audit/export permissions but does not include the mapped `withdrawals.approve` permission. The high-value approval tier (>= $50,000) lists Compliance as an eligible role and requires Finance or Compliance participation, so Compliance can satisfy the tier-role helper but cannot currently pass the action-permission gate for a WITHDRAWAL request.

This was intentionally not resolved by silently broadening Compliance authority. The next Admin control-plane pass should make a deliberate policy decision: either extend the permission catalog to grant the precise approval capability or make the approval policy explicitly tier-aware. Until that decision is made, FINANCE_ADMIN is the operational route for withdrawal approvals.

## Scope note

The PR changed two files and exceeded the preferred 500-line review target because the regression suite intentionally includes both mocked state-machine tests and live Postgres concurrency proofs. No schema, route, or duplicate service authority was introduced.

## Next engineering target

Continue the Admin control-plane audit beyond this approval slice: inspect privileged mutation endpoints for effective permission enforcement, state-machine/CAS safety, auditability, and actor identity integrity. Then continue the roadmap into broader tenant/state convergence and production-readiness work.
