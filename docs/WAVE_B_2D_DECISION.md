# Studio Wave B — 2D Surface Decision

**Decision (2026-09-05):** Do not restore the historical 2D canvas solely to satisfy reopened magnetic-snap criteria.

Current V2 is a semantic layer-tree editor whose central stage mounts `StorefrontPhonePreview`. The legacy `StorefrontCanvas.jsx` owns the historical grid geometry and magnetic snap engine but is not mounted by V2. Reintroducing it now would create competing layout authorities.

Keep Pointer Events palette insertion as the current V2 drag acceptance. Keep the semantic document as persistence authority. Revisit 2D only when a real product requirement needs spatial placement that the semantic layout model cannot express.

Before any future 2D implementation, define the canonical coordinate model, responsive behavior, persistence/mutation boundary, renderer/interaction ownership, and executable snap/fuse/resize/keyboard/settle contracts. Only then adapt the existing drag engine or replace it.
