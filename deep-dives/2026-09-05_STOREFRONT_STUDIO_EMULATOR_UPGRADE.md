# Storefront Studio Emulator Upgrade — M3E Canvas Alignment

**Date:** 2026-09-05
**Status:** PROPOSED — awaiting merge into the active loop
**Reference studied:** https://github.com/lnkiai/m3e-canvas (MIT) — Material 3 Expressive screen sketcher with drag-and-drop, magnetic snap, tap-through preview, tidy layout, layers panel
**Author:** Lyra (lead engineering review)

## Purpose

The Business Portal Storefront Studio (StorefrontStudioV2 + StorefrontPhonePreview) has the right architecture: a semantic tree editor, one pure runtime adapter to the customer-facing widget contract, and a real test suite. What it lacks is editor craft: dragging feels like HTML5 ghosts, the preview is a static drawing with hand-guessed dimensions, and layout correctness is by eyeball.

This document defines a phased upgrade modeled on m3e-canvas's proven patterns, adapted to our semantic-tree architecture.

## Explicit non-goals — do NOT build

- **No AI generation of any kind.** No "describe your store and it gets generated", no prompt compiler, no BYO-API-key AI helper, no behavior-note writer. The Studio stays deterministic and human-driven. The owner has explicitly rejected this direction.
- **No free-canvas geometry model.** m3e-canvas is a free-position sketcher; our Studio is a semantic tree whose adapter output is the customer runtime contract. We port patterns (tokens, snap physics, tidy rules), not their coordinate system.
- **No localStorage persistence.** We already persist drafts through the backend; nothing changes there.
- **No rebranding of the Experience Blueprint policy layer.** This upgrade is strictly the Studio editor/preview surface.

## What we keep unchanged

- `storefrontStudioRuntimeAdapter.js` as the single bridge from tree to runtime tiles. Every wave below must keep the adapter contract tests green.
- Backend as the sole authority for storefront draft/publish state.
- The existing Studio test suite (keyboard movement, drop insertion, editor selection, sanitization, collision).

---

## Wave A — Token foundation (do first, no behavior change)

**Problem:** `StorefrontPhonePreview.jsx` hardcodes visual constants inline (fontSize 11, heightMap 60/80/100, guessed paddings). Every renderer drifts independently from the Flutter runtime.

**Change:** Create `src/lib/storefrontStudioTokens.js` — one frozen module holding:

- device geometry: phone 412x892 (dp-true, not the current CSS guess), tablet, desktop widths, corner radii, and the "expanded" breakpoint where responsive layout switches
- layout margins (16dp default), gaps, gutters
- type scale (one named scale, no inline pixel literals)
- snap physics constants: snap distances, pull sharpness, settle animation duration (~340ms)
- shape/spacing tokens the runtime adapter also consumes

Refactor the mini-widget renderers, `storefrontStudioResponsive.js`, and the viewport switcher to read from it. **Zero visual behavior change intended in this wave.**

**Acceptance:** every existing test passes unmodified; no inline px literals remain in `StorefrontPhonePreview.jsx` except token lookups; adapter contract tests green.

## Wave B — Pointer-based drag with magnetic snap

**Problem:** Studio uses HTML5 drag events. No touch support, ghost images, no physics, drops don't feel intentional.

**Change (modeled on m3e-canvas's pointer implementation):**

- pointerdown on a tile takes pointer capture with window-level pointermove/pointerup/pointercancel listeners
- magnetic snap fields: along-sibling and cross-sibling snap distances with a pull curve (gentle at the edge, sharpening near the target)
- tiles of the same family dropped adjacent **fuse into a connected group** with softened inner corners — list items stack, buttons row
- settle animation on landing (~340ms) before the change is committed to the draft
- drop edges highlight with an insertion caret, as today, but resolved through the pointer geometry
- touch input works identically (pointer events give this for free)

**Acceptance:** pointer-simulation tests for capture, snap, fuse, settle-commit; keyboard movement tests still pass; no HTML5 drag attributes remain on Studio surfaces.

## Wave C — Real device emulation

**Problem:** the preview is a fixed-height drawing. Viewport switching only changes grid columns. It cannot scroll, so it cannot catch layout bugs.

**Change:**

- real device frames from Wave A tokens: phone, tablet, desktop; status/nav insets drawn as part of the frame
- scrollable content inside the frame (real overflow, real scroll position)
- viewport switch re-lays out the document against the responsive module — bars stretch, nav-style elements adapt at the expanded breakpoint
- density/safe-area handling so the preview matches what Flutter renders on a real device

**Acceptance:** preview scrolls; viewport switch visibly re-lays the same draft; a layout bug (overlapping tiles, clipped text) is demonstrable in the preview.

## Wave D — Tap-through navigation preview

**Problem:** the design is not playable. The merchant cannot feel the customer journey inside the Studio.

**Change:**

- a **Preview mode** (distinct from Edit mode) where tappable tiles navigate between Studio pages: product grid to product detail page, button to target page, back returns with reversed transition
- per-tile navigation target + transition type stored in the node props (slide/fade/expand/none)
- flow arrows between linked pages on the canvas (m3e-canvas pattern)
- transitions animate in Preview mode only; Edit mode stays instant

**Note:** this is page-to-page navigation inside the Studio document, NOT live route execution. It stays deterministic and preview-only.

**Acceptance:** a two-page draft can be tapped through in Preview with transitions and back; targets survive save/load of the draft.

## Wave E — Tidy pass and alignment guides

**Problem:** hand-placed trees accumulate 3px misalignments and inconsistent gaps, and nothing normalizes them.

**Change (modeled on m3e-canvas tidy.ts — pure, rule-based, fully unit-tested):**

- a pure `tidyStudioDocument(document)` function: near-aligned edges snap at ~6dp threshold, gaps normalize to the token margin scale, same-family adjacent siblings join, a part keeps the side it was on, sizes/order/content never change
- one **Tidy button**, the operation is one undo step
- alignment guides during drag: when a dragged edge aligns within threshold with a sibling edge or margin, draw a snap line
- the tidy function must not "guess" — it only fixes differences a hand would call a slip (their rule, adopt it verbatim)

**Acceptance:** tidy has its own unit test suite covering each rule and its thresholds; one-click undo restores exactly.

## Wave F — Layers panel upgrade

**Change:** the Studio layer tree gains m3e-canvas layers behavior: drag a row to reorder z-priority among siblings, open a group to reorder children inside it, and the row selection syncs bidirectionally with canvas selection.

**Acceptance:** reordering via layers panel is undoable and reflected in the adapter output order.

---

## Sizing and sequencing for the active loop

- One wave = one or two PRs, each under the ~500-line diff rule, each with tests.
- Wave A first (it de-risks every later wave). B and C next. D and E after. F last.
- Every PR keeps `storefrontStudioRuntimeAdapter` contract tests green — the adapter is the customer-facing guarantee and must never be edited to accommodate editor convenience.
- Waves B–D touch `StorefrontPhonePreview.jsx` heavily; consider splitting the file per widget renderer during the Wave A refactor only if it stays under the diff cap.

## Why this matters (for prioritization)

The Storefront/Experience Studio is the merchant-facing product surface that makes AZAMAN sellable as a commerce OS. It is currently the weakest felt-quality surface while being one of the best-tested. This upgrade converts it from "functional editor" to "editor merchants enjoy using", with no new authority, no new backend surface, and no AI anywhere.

---

## Appendix — verified Flutter runtime constants (measured 2026-09-05)

Source: `AZM-frontend/lib/storefront/` — the registry (`core/storefront_widget_registry.dart`) has **16 widget types**; the Business Portal preview (`StorefrontPhonePreview.jsx`) covers **15**. `retail_collection_box` renders as the dashed FallbackTile in the Studio — merchants currently see a placeholder for a real Nitro widget. Fix as part of Wave A.

### Drift measured against the real runtime

| Constant | Flutter (real, dp) | Business Portal preview (CSS px) | Ratio |
|---|---|---|---|
| Hero height compact | 140 | 60 | 2.33x |
| Hero height standard | 200 | 80 | 2.50x |
| Hero height tall | 260 | 100 | 2.60x |
| Hero title fontSize | 24 | 11 | 2.18x |
| Hero subtitle fontSize | 14 | 9 | 1.56x |
| QuickInfoBar padding | 16h / 10v | 6 / 10 | — |
| QuickInfoBar label fontSize | 12 | 9 | 1.33x |
| Product grid spacing | 10 / 10, aspect 0.75 | guessed per-renderer | — |

The scale factors are inconsistent per widget — the preview is not a miniaturization of the real app, it is a different layout. Proportions lie, which is worse than being small.

### Wave A grounding rule

- `storefrontStudioTokens.js` must hold **the Flutter dp values as the source of truth** (hero 140/200/260, title 24, subtitle 14, quick-info 12, grid 10/10 aspect 0.75, hero radius 12, etc.) plus **one explicit `PREVIEW_SCALE` factor** applied uniformly for the canvas.
- Extract remaining constants by reading each `lib/storefront/widgets/*_widget.dart` before writing its token — never estimate.
- Add `retail_collection_box` to the Business Portal `WIDGET_RENDERERS` map so preview coverage reaches 16/16.
- The responsive/viewport module and the runtime adapter read the same tokens; adapter output shape is unchanged (tokens affect Studio preview only, not the customer runtime contract).
