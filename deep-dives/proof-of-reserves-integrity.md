# Proof-of-Reserves Integrity Deep Dive — 2026-08-30

## Why this batch exists

The existing Proof-of-Reserves implementation had a historical-verification flaw: a snapshot stored a Merkle root and salt, but `/verify` reconstructed the user's leaf from the user's **current** balances. Once a user balance changed, an otherwise valid historical snapshot could no longer prove the balance represented at snapshot time.

There was also a lifecycle problem: the public `GET /api/proof-of-reserves` endpoint created and persisted a new snapshot on every read. A public information request therefore mutated financial history and could create unbounded snapshot rows.

## Implemented correction

The backend `financial-truth-reconciliation-batch` now introduces an immutable `ProofOfReservesLeaf` table containing, per snapshot and user:

- available balance at snapshot time;
- escrow-locked balance at snapshot time;
- vendor-unallocated balance at snapshot time;
- dispute-escrow balance at snapshot time;
- the salted leaf hash used in the Merkle tree.

Snapshot creation and leaf persistence happen inside one Serializable database transaction.

The public read path now returns the latest snapshot without mutation. Snapshot creation is an explicit admin operation and is also scheduled hourly by the backend worker registry.

User verification reads the immutable historical leaf and reconstructs the Merkle proof from stored leaf hashes. This makes verification independent of the user's current live balance.

## Financial integrity surface

`GET /api/journal/integrity` now combines:

1. double-entry journal trial-balance status;
2. latest Proof-of-Reserves snapshot;
3. snapshot leaf coverage;
4. reserve/liability totals;
5. Merkle root.

An integrity exception returns HTTP 409 rather than being presented as a successful health result.

## Verification added

Merkle invariants are covered for:

- empty tree;
- deterministic root generation;
- odd-leaf duplication;
- multiple leaf positions;
- tampered leaf rejection;
- proof-index binding.

## Architectural consequence

This moves AZAMAN closer to the master financial-truth goal:

`authoritative balances → immutable snapshot → cryptographic commitment → verifiable user proof → admin integrity signal`

It does not by itself prove external bank/blockchain reserves. The current reserve calculation remains based on AZAMAN's configured system reserve pools. External custody reconciliation remains a separate P0 requirement.

## Related implementation

- `AZM-backend/services/proofOfReservesIntegrityService.js`
- `AZM-backend/routes/proofOfReservesRoutes.js`
- `AZM-backend/routes/journalRoutes.js`
- `AZM-backend/workers/proofOfReservesWorker.js`
- `AZM-backend/src/workers/index.js`
- `AZM-backend/prisma/migrations/20260830070000_proof_of_reserves_leaf/migration.sql`

## Status

**IMPLEMENTED ON BRANCH / CI PENDING**

The batch is intentionally kept together until the complete backend gate validates migration, service, routes and worker registration as one unit.
