---
name: phaser4-runtime
description: Phaser 4 runtime specifics plus v3-contamination warnings — the details of building the game runtime on Phaser 4 and the traps that arise when Phaser 3 knowledge or code leaks in. Consult this skill whenever working with the Phaser 4 runtime, wiring the editor's output to Phaser, or checking whether a Phaser pattern is v4-correct rather than stale v3 habit. Targets Phaser 4.x.
---

# phaser4-runtime

The runtime skill for Phaser 4 — the specifics of building the game runtime on Phaser 4 and, just as importantly, the warnings about Phaser 3 patterns that no longer apply and cause bugs when they contaminate v4 work. This skill is version-pinned: its guidance targets Phaser 4.x, and version-dependent advice is marked as such. Ground-truth Phaser 4 docs live under `vendor/`. Content is earned from real runtime sessions, never invented.

## Decisions

_None recorded yet. Filled via dual-write as the kernel is built._

## Gotchas

### G1: v3 contamination — AI-generated Phaser code defaults to Phaser 3

Model training data is saturated with a decade of Phaser 3 tutorials and Stack Overflow answers. Any session writing Phaser code from memory will produce v3 idioms, and most of them fail in v4 — some loudly, some silently. Before trusting any generated Phaser API call, check it against `vendor/phaser4/MIGRATION-GUIDE.md` (the authority for what changed) and `vendor/phaser4/types/phaser.d.ts` (the authority for what exists).

The silent failures are the dangerous ones — same name, different behavior, no error:

- `Math.TAU` **changed value**: PI/2 in v3, PI*2 in v4. Rotation code contaminated with the v3 assumption is wrong by a factor of 4 and throws nothing. v3-`TAU` intent now needs `Math.PI_OVER_2`.
- `roundPixels` default flipped `true` → `false`. Pixel-art rendering silently blurs.
- `DynamicTexture`/`RenderTexture` draw commands are now buffered; nothing appears until `render()` is called. v3 habit executes draws immediately and expects them visible.

Loud failures a v3-trained session will still write confidently: `setTintFill()` (gone → `setTint().setTintMode(Phaser.TintModes.FILL)`), `Geom.Point` (gone → `Vector2`), `setPipeline('Light2D')` (pipelines removed entirely → `setLighting(true)`), `BitmapMask` (gone → `filters.internal.addMask()`), `Phaser.Struct.Set`/`Map` (gone → native `Set`/`Map`), `Mesh`/`Plane` game objects (removed).

**Fix/policy:** never write Phaser code purely from memory. Verify against the vendored `.d.ts`; when editor or runtime code touches an area listed in the migration guide's checklist, read that section first. Phaser also ships official per-topic skills inside its npm package (see `vendor/phaser4/PROVENANCE.md`) including a v3→v4 migration skill vendored at `vendor/phaser4/v3-to-v4-migration-SKILL.md`.

_(Recorded 2026-08-11 from the vendored 4.1.0 migration guide, at vendoring time — before kernel runtime sessions began. Session-earned contamination traps append below as they are hit.)_

## Contracts

_None recorded yet. Reference kernel-repo and `vendor/` Phaser 4 doc file paths here; never paraphrase them as prose._
