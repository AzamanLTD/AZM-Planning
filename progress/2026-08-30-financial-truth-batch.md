# Financial Truth Batch — 2026-08-30

## Batch scope

This is a coherent P0 financial-truth hardening batch, not a cosmetic test cleanup.

## Implemented in backend PR #29

- Immutable per-user Proof-of-Reserves commitments are persisted per snapshot.
- Snapshot + all leaves are created in one Serializable transaction.
- Historical user verification uses the balance state committed at snapshot time.
- Public Proof-of-Reserves GET is read-only and cannot create unbounded snapshots.
- Snapshot creation is an explicit admin mutation (`POST /api/proof-of-reserves/refresh`).
- Hourly Proof-of-Reserves snapshot worker registration has been added to the distributed scheduler.
- Admin journal integrity endpoint combines double-entry trial balance and PoR commitment coverage.
- Integrity is exceptional when the journal is unbalanced, the snapshot is incomplete, reserves are under-backed, or the latest snapshot is older than two hours.
- Public PoR returns an explicit unavailable response when no snapshot exists rather than fabricating a successful state.
- Merkle regression tests cover deterministic roots, odd leaves, multiple positions, tampering, and index binding.
- E2E cleanup is scoped to the smoke-test users and no longer calls the nonexistent `businessOrderItem` Prisma model.

## Important architecture corrections

The old PoR flow calculated a Merkle root over a point-in-time user list but `/verify` reconstructed leaves from live balances. Historical verification therefore became invalid after a balance change. Immutable snapshot leaves now close that gap.

The old public endpoint also created a database snapshot on every GET. Snapshot creation is now separated from public reads and runs operationally on the scheduler.

## Deliberate boundaries

This batch does **not** claim that AZAMAN's internal reserve-pool balances are proof of external bank/custodian assets. External custody reconciliation remains a separate P0 requirement.

It also does not pretend that every legacy financial operation is journaled. The integrity surface explicitly exposes journal imbalance rather than silently repairing it.

## Planning artifacts updated in the same batch

- `deep-dives/proof-of-reserves-integrity.md`
- `AZM architecture visual maps.md`
- this progress record

## Verification status

Backend PR #29 is intentionally open until the complete backend gate validates migration, generated Prisma client, service, routes, worker registration, E2E cleanup, and regression tests together.

## Next whole-system financial-truth direction

`financial operation identity → authoritative mutation → journal entry → external provider state → reconciliation exception → admin queue → auditable repair`

The next engineering batch should connect financial operation identity and provider callbacks into this integrity surface so discrepancies become durable, reviewable objects instead of logs.

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
