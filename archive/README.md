# AZAMAN Planning Archive

This directory is for historical assessments, dated handoffs, progress logs and deep dives that are useful for context but must not compete with the active engineering plan.

## Authority order

1. Current repository `main` + exact CI evidence.
2. `START_HERE.md`.
3. `ROADMAP.md`.
4. `CURRENT_STATE.md`.
5. Living architecture/contract/invariant/state/release documents.
6. Historical material in this archive.

Historical documents are preserved for reasoning and provenance. If a historical finding remains relevant, copy the **fact and its current verification status** into the appropriate living document rather than asking future agents to rediscover it from an old journal.

## Legacy root documents

The repository previously accumulated dated root-level documents and agent-specific plans. They are historical. During this reorganization they should either be moved here in a content-preserving rename, or left in Git history when moving them would add no useful searchable copy. Never treat their timestamps as current implementation evidence.

## Archive rule

No new session-journal files should be created. A substantial session updates `CURRENT_STATE.md`; architectural facts update the relevant living document; deep research can be preserved here when it has durable value.
