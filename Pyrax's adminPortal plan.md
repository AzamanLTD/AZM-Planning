# Pyrax's Admin Portal Plan

**Repository:** `AzamanLTD/AZM-adminPortal`  
**Status:** IN PROGRESS / ARCHITECTURE MAPPING  
**Last updated:** 2026-08-30 (UTC)

## Mission

The Admin Portal is the platform-control surface. It exists to operate AZM safely at scale: users, businesses, permissions, financial oversight, disputes, risk, moderation, configuration, observability and support.

It must be more powerful than merchant/customer surfaces while remaining constrained by explicit authorization and audit rules.

## Master architecture role

`Admin identity → privileged authorization → audited administrative action → backend domain service → authoritative state → event/audit trail`

Administrative UI must never directly mutate database truth outside supported backend contracts.

## Core responsibilities

- platform user/business oversight
- role and permission administration
- KYC/KYB and trust workflows where applicable
- payment/escrow oversight
- refunds/disputes/reconciliation operations
- suspicious activity/risk investigation
- moderation and reports
- notification/platform configuration
- operational health and incident response
- audit-log inspection
- controlled feature/configuration management.

## Financial safety

Admin tooling must make financial authority and auditability explicit. High-impact actions should require appropriate permission levels, record actor/reason, preserve before/after state where safe, and be idempotent where repeated submission is possible.

The Admin Portal must not become a backdoor around customer/business authorization. “Admin” is a role with defined capabilities, not an excuse to bypass all controls silently.

## State and event architecture

Administrative views may consume realtime events, but critical state is fetched from authoritative APIs. Sensitive actions should produce durable audit events after successful state transitions.

## Social/moderation

The platform's social graph will eventually require:

`report → moderation queue → investigation → action → audit → notification`

Moderation must distinguish content/community actions from financial actions. Removing a post should not implicitly alter a transaction; refunding a transaction should not implicitly delete social history unless a documented policy says so.

## Observability

The Admin Portal should eventually expose operational views for:

- request/error rates
- payment/escrow mismatches
- order-state anomalies
- inventory anomalies
- event delivery failures
- webhook/replay anomalies
- reconciliation discrepancies
- abuse signals
- platform incidents.

Sensitive information must be minimized and role-scoped.

## Current status

### VERIFIED / KNOWN

- Backend contains the authoritative service layer used by customer/business experiences.
- Financial/order integrity work is being hardened in the backend.
- Social/follow foundations exist and will eventually require moderation/oversight.

### IN PROGRESS

- Mapping existing Admin Portal screens and API contracts.
- Identifying privileged actions and their required audit boundaries.
- Mapping payment, escrow, dispute and reconciliation workflows.

### PLANNED

- Unified operations dashboard.
- Financial reconciliation console.
- Risk/fraud investigation workflow.
- Moderation queue and policy tooling.
- Comprehensive audit explorer.
- Role/permission management.
- Platform-wide health/event monitoring.

## Verification

Privileged actions require authorization tests, audit verification, negative tests and concurrency/idempotency checks where applicable. Never verify admin functionality solely by checking that a button works in the UI.

## Agent continuation rules

1. Read this file before Admin Portal changes.
2. Trace each privileged action to its backend authorization and state transition.
3. Never weaken authorization to make UI workflows easier.
4. Require auditability for sensitive administrative actions.
5. Keep financial actions separate from moderation/social actions.
6. Record implementation versus verified status honestly.
7. Update this plan whenever architecture or privileged workflows change.
