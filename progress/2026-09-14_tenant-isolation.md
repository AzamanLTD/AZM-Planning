# 2026-09-14 Business OS tenant-isolation audit — verified remediation

## Finding
`services/businessOS/businessGroupService.js::getGroupStats(userId, groupId)` previously queried `BusinessProfile` with `{ groupId }` when a group was supplied. The service did not first verify that the requested `BusinessGroup` belonged to the calling `userId`, so possession/guessing of another group's UUID could expose that group's business aggregates.

## Remediation implemented
`AZM-backend` PR #239 now:
1. verifies `BusinessGroup.id` together with `ownerUserId = userId`;
2. fails closed with an empty aggregate for a foreign/nonexistent group;
3. constrains the subsequent `BusinessProfile.findMany` to both `groupId` and `userId`.

No controller wrapper or alternate authorization authority was introduced; the ownership boundary remains in the canonical Business Group service.

## Regression coverage
Added `__tests__/business-group-service-tenant-isolation.test.js` covering:
- foreign group ID does not trigger a business-profile query;
- valid owned group queries only the caller's businesses.

## Verification
- PR: `AzamanLTD/AZM-backend#239`
- Exact CI head: `0fc7d575234021c5e4d31a0e7808d77805c27db7`
- Backend CI run `#917` passed the full canonical pipeline: dependency install, production dependency audit, database schema application, Prisma generation, full Jest suite, route registry verification, and database backup/restore drill.
- Merged to backend main as `8421eaa69bb03f5c4e4251491d00f0e8e301a7a1`.

## Supporting schema evidence
`BusinessGroup` has an explicit `ownerUserId` foreign key/index, while `BusinessProfile.groupId` links businesses to that group. The security invariant is therefore owner → group → business, not group UUID alone.

## Broader audit notes
`businessOrderService` was reviewed as a neighboring tenant boundary. Its owner-facing controller first resolves the authenticated user's `BusinessProfile` and passes that scoped ID to listing/stats operations; order-detail access explicitly checks either business ownership or customer identity before returning the order. No additional immediate P0 was opened from that path during this loop.

`businessLocationService` similarly requires a resolved `businessProfileId` for private mutations/listing and verifies location/table ownership before update/delete operations. A separate concurrency hardening opportunity remains around the max-location count / primary-location invariant but is not a tenant data-exposure P0.
