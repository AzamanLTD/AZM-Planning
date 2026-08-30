# Financial Truth Batch — 2026-08-30

## Batch scope

This is a coherent P0 financial-truth hardening batch, not a cosmetic test cleanup.

### Backend implementation in progress

- Immutable per-user Proof-of-Reserves commitments are persisted per snapshot.
- Snapshot + all leaves are created in one Serializable transaction.
- Historical user verification uses the balance state committed at snapshot time.
- Public Proof-of-Reserves GET is now read-only and cannot create unbounded snapshots.
- Snapshot creation is exposed as an explicit admin refresh operation.
- Hourly Proof-of-Reserves worker registration has been added to the distributed scheduler.
- Admin journal integrity endpoint combines journal trial balance and PoR commitment coverage.
- Financial integrity exceptions return a machine-visible conflict rather than a false success.
- Merkle proof tests cover deterministic roots, odd leaves, multiple positions, tampering, and index binding.

## Important architecture correction

The old PoR flow was useful as a solvency display but did not provide a durable historical proof. It calculated a root over a point-in-time user list, while `/verify` reconstructed the leaf from live balances. This is now corrected.

The public endpoint also previously generated a database snapshot on every GET. This is now separated into read and mutation paths.

## What this does NOT claim

- It does not prove external bank/custodian balances.
- It does not reconcile every historical financial operation to the double-entry journal.
- It does not make legacy non-journaled operations magically journaled.

Those remain explicit P0 financial-truth work rather than being marked complete prematurely.

## Current repository evidence

Backend branch: `financial-truth-reconciliation-batch`

Planning documents updated:

- `deep-dives/proof-of-reserves-integrity.md`
- `AZM architecture visual maps.md`
- this progress record

## Next coherent financial-truth direction

After this batch is verified, continue from:

`financial operation identity → authoritative mutation → journal entry → external provider state → reconciliation exception → admin queue → auditable repair`

Do not substitute dashboards for reconciliation and do not hide discrepancies with automatic state rewriting.
