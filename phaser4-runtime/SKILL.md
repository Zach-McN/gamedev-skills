---
name: phaser4-runtime
description: Phaser 4 runtime specifics plus v3-contamination warnings — the details of building the game runtime on Phaser 4 and the traps that arise when Phaser 3 knowledge or code leaks in. Consult this skill whenever working with the Phaser 4 runtime, wiring the editor's output to Phaser, or checking whether a Phaser pattern is v4-correct rather than stale v3 habit. Targets Phaser 4.x.
---

# phaser4-runtime

The runtime skill for Phaser 4 — the specifics of building the game runtime on Phaser 4 and, just as importantly, the warnings about Phaser 3 patterns that no longer apply and cause bugs when they contaminate v4 work. This skill is version-pinned: its guidance targets Phaser 4.x, and version-dependent advice is marked as such. Ground-truth Phaser 4 docs live under `vendor/`. Content is earned from real runtime sessions, never invented.

## Decisions

### P1: Pin Phaser to the version the vendored docs came from, exactly

`"phaser": "4.1.0"`, no caret, matching `vendor/phaser4/PROVENANCE.md`. **Reason:** this skill names the vendored files as the authority and tells every session to check them rather than trust memory (G1). If the installed version drifts ahead of the vendored one, that instruction becomes false in a way nobody notices — the session dutifully reads the docs, gets an answer that was right for 4.1.0, and writes code against a library that has moved. A caret makes that happen silently on somebody else's `npm install`. Upgrading is a deliberate act: bump the pin and re-vendor in the same change, never one without the other. _[earned 2026-08-11]_

### P2: One game for the life of the window, told what to show

A single `Phaser.Game`, booted against a canvas the host supplies, with a small imperative API over it — show this, redraw it at that scale, resize, clear. Never one game per thing being previewed. **Reason:** three costs, of which only the first is obvious. Booting is slow enough to see. Each boot takes a fresh WebGL context and browsers cap how many exist at once (around sixteen in Chromium), after which the oldest are silently killed — so a long session starts losing contexts for reasons that look like anything but "we made too many". And a rebuild throws away everything the renderer had cached, which turns the operations that should be free (changing a filter, re-cutting frames) back into first loads. The API this forces is about fifty lines and is the good kind of constraint: it makes "what can change without a reload?" an explicit, reviewable list. _[earned 2026-08-11, Phaser 4.1.0]_

**Amended when a second surface arrived: one game per *declared surface*, not one game per window.** The editor now has two — a scene viewport and a single-texture preview — and that is fine, because none of P2's three costs is about the number two. They are all about a count that grows with *use*: booting where the human can see it, contexts churning until the browser kills the oldest, and caches thrown away that should have been warm. The invariant that actually protects against those is **the renderer count is a number written in one file** (`editor-ui` U18, `panels.tsx`) rather than a number nobody can predict. Two panels, two games, each booted once and kept for the life of the window.

Two things learned running two of them, both reassuring: nothing in Phaser is global between games — the same scene key, the same texture keys and separate `TextureManager`s coexist without a word — and the second game costs one more WebGL context and nothing else. The rule to keep is the *reviewability* one: if a session ever finds itself creating a game somewhere other than a provider that owns exactly one, that is the moment P2 is actually being broken, whatever the current count happens to be. _[earned 2026-08-11, Phaser 4.1.0]_

### P3: Import settings are applied to the live texture, never by reloading it

`Texture.setFilter(mode)` changes filtering in place. Frames are replaced with `texture.remove(name)` for each existing frame and `texture.add(name, 0, x, y, w, h)` for each new one. Neither touches the network and neither re-uploads the image to the GPU — frames are rectangles held in JavaScript, so re-slicing is arithmetic. **Reason:** the difference is invisible on a 16px sprite and decides whether the tool is usable on a real one. Dragging a frame-size field on a 4096² tileset is a ~64MB upload per keystroke if the texture is re-registered, and free if it is not. `Texture.get('__BASE')` survives slicing and is still the whole image, which is what lets a preview draw the sheet and its frame boundaries at the same time. _[earned 2026-08-11, Phaser 4.1.0]_

### P4: Report what the texture is doing, read back off the texture

Anything that says what the renderer is up to — a status attribute, a test hook, a debug readout — reads `texture.source[0].scaleMode` rather than echoing the value that was handed to `setFilter`. **Reason:** a report built from the request keeps saying the right thing long after the line that applies it stops working, so the one assertion that could have caught the regression is the one that cannot. This is also what makes a canvas feature testable without comparing pixels: the renderer is the witness rather than the caller. _[earned 2026-08-11, Phaser 4.1.0]_

### P5: Pixel art is kept crisp by moving the camera onto the device grid, never by rounding each sprite

`roundPixels` defaults to `false` in v4 (G1), and the temptation is either to switch it back on or to `Math.round` every sprite's position on the way into `setPosition`. Do neither. Adjust the *camera* by less than a pixel so that the scene's origin lands on a whole device pixel, and position everything through that camera unrounded.

**Reason:** all three keep the art crisp; only this one keeps the distance between two sprites exact. Per-sprite rounding moves each object by its own fraction of a pixel, which is invisible on screen and not invisible at all to anything that *measures* what was drawn — the extent of a level then comes out slightly different at each zoom, and a "frame everything" built on it stops being idempotent. That surfaces two files away as a key that does something different the second time it is pressed.

Two practicalities. The residue is honest and worth stating in the code: an entity at a fractional position is still drawn at a fractional position and is still soft, which is the designer's own number rendered faithfully. And whatever the renderer *reports* should be the camera it was handed rather than the nudged one — the snap is sub-pixel presentation, and reporting it makes the caller's own state disagree with itself the next time it compares. _[earned 2026-08-12, Phaser 4.1.0]_

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

### G2: The sprite-sheet parser is not public in v4, and that is a better arrangement than it looks

v3 exposed `Phaser.Textures.Parsers.SpriteSheet`, which a v3-trained session will reach for to re-cut a texture into frames. In 4.1.0 the `Phaser.Textures.Parsers` namespace contains only compressed-texture helpers — `KTXParser`, `PVRParser`, `PCTDecode`, `verifyCompressedTexture` — and nothing for sprite sheets. `TextureManager.addSpriteSheet(key, source, config)` exists but mints a *new* texture from a source, which is the wrong shape for changing the slicing of one already on screen.

**Fix/policy:** compute the frame rectangles yourself and add them with `Texture.add`. Do not treat this as a workaround. The frame geometry — what `frameWidth: 24, margin: 2, spacing: 1` means on an image of a given size — becomes a pure function you own, which is testable in Vitest with no browser, no canvas and no renderer, and which the editor and the runtime can then *share* rather than each interpreting the settings their own way. That sharing is the thing that makes a frame guide drawn over a preview honest: it is describing the same arithmetic that cut the frames.

The arithmetic has one edge worth stating, because it is easy to get subtly wrong and the wrongness is invisible. `n` frames along an axis need `2*margin + n*frame + (n-1)*spacing` pixels — the gaps go *between* frames, so there are `n-1` of them, not `n`. And a grid that fails on one axis produces no frames at all, so it covers nothing on *both*: reporting "no columns but one row" describes a row of frames that does not exist, and anything drawing the leftovers then shades a strip down one side of an image where every pixel is a leftover. _[earned 2026-08-11, Phaser 4.1.0]_

### G3: `export = Phaser` needs a namespace import, and the ESM build is fine with it

The vendored `phaser.d.ts` ends with `declare module 'phaser' { export = Phaser; }`. Under `verbatimModuleSyntax` without `esModuleInterop` — the strict setup this kernel uses — `import Phaser from 'phaser'` does not compile. **Fix/policy:** `import * as Phaser from 'phaser'`. Do not reach for `esModuleInterop` to make the default import work: it is a whole-project compiler change made to accommodate one package, and it is not needed. The npm ESM build (`dist/phaser.esm.js`) carries named exports for everything reachable off the namespace (`Game`, `Scene`, `Textures`, `Scale`, `WEBGL`, …), so the namespace import is correct at runtime as well as at compile time. _[earned 2026-08-11, Phaser 4.1.0, TypeScript 5.9.3]_

### G4: `create` is not declared on `Phaser.Scene`, so it must not carry `override`

The `Scene` class in the v4 types declares `update(time, delta)` but not `create`, `preload` or `init` — they are hooks the framework calls by name. Under `noImplicitOverride` this is the opposite of the usual trap: writing `override create()` is the error, and plain `create()` is correct. **Fix/policy:** worth knowing on sight, because the compiler message points at the keyword rather than at the base class and reads as though the method is misspelt. _[earned 2026-08-11, Phaser 4.1.0]_

### G5: A game given `parent: undefined` appends its canvas to the body

`parent` accepts an element, an id, `null` or nothing — and nothing means "put the canvas in `document.body`". A renderer whose canvas is meant to be handed to whichever panel is currently hosting it must pass `parent: null` explicitly, or a stray canvas appears at the end of the page. **Fix/policy:** `parent: null` whenever the host owns placement, and set the canvas's CSS size yourself. Pair it with `scale: { mode: Phaser.Scale.NONE }` so the Scale Manager does not start watching the window and resizing behind the host's back. _[earned 2026-08-11, Phaser 4.1.0]_

### G6: Headless Chromium under Playwright has real WebGL 2, so a canvas feature is testable in CI

Measured rather than assumed, because the opposite is widely believed: a default `chromium.launch()` reports `WebGL 2.0 (OpenGL ES 3.0 Chromium)` on an ANGLE/SwiftShader device with an 8192px maximum texture size. No launch flags, no `--use-gl=swiftshader`, no `--enable-unsafe-swiftshader`. **Fix/policy:** do not add GL launch arguments speculatively; probe first, in five lines, and add them only if the probe fails. The 8192 limit is the one number worth remembering — a test fixture larger than that would fail for a reason that looks nothing like its cause. _[earned 2026-08-11, Playwright 1.62.1]_

### G7: After import settings are applied, frame `"0"` exists unless the grid produced nothing

`applyImportSettings` removes every frame name and re-adds one per rectangle, and `framesFor` returns a single whole-image rectangle for `single` — so a texture that has been through it always has a frame called `"0"`, whatever its slice mode. The exception is a grid whose frame size does not fit the image at all, which produces no frames, and then only `__BASE` is left.

**Fix/policy:** a sprite draws `texture.has('0') ? '0' : '__BASE'`. Do not reach for `__BASE` as the normal case — it is the whole sheet, so a four-frame run strip would draw as a wide smear rather than as a character, and it would look like a scaling bug. Do not assume `'0'` either: the fallback is the case where a human has typed a frame size larger than their image, which is ordinary while typing. _[earned 2026-08-11, Phaser 4.1.0]_

### G8: `getBounds()` is correct the moment a game object is positioned, with no render tick in between

Setting origin, position, rotation, scale and depth and then reading `getBounds()` in the same synchronous block gives the rectangle the object will actually occupy. **Fix/policy:** this is what lets a renderer *report* what it drew rather than have a panel recompute it from the same inputs (`editor-kernel` D2) — the report is available immediately, so there is no excuse for a second derivation. Divide by the device pixel ratio on the way out if the object was positioned in device pixels; the bounds come back in the same space the transform was set in, not in CSS pixels. _[earned 2026-08-11, Phaser 4.1.0]_

## Contracts

Contracts are referenced as file paths, never paraphrased as prose. Read the file; don't trust a summary of it.

- `gamedev-skills/vendor/phaser4/PROVENANCE.md` — which Phaser version the vendored ground truth came from. The pin in `kernel-2d/package.json` must match it (P1).
- `gamedev-skills/vendor/phaser4/MIGRATION-GUIDE.md` — the authority for what changed between v3 and v4.
- `gamedev-skills/vendor/phaser4/types/phaser.d.ts` — the authority for what exists.
- `kernel-2d/runtime/index.ts` — the runtime's public surface: what the editor is allowed to import, and the one-way arrow that keeps an exported game clean.
- `kernel-2d/runtime/preview/texture-view.ts` — P2 and G5 as built: booting one game against a supplied canvas, the image cache, and the report of what was actually drawn.
- `kernel-2d/runtime/textures/import-settings.ts` — P3 and P4: settings applied to a live texture, and the filter read back off it.
- `kernel-2d/runtime/textures/frames.ts` — the frame geometry of G2. The single definition of what a slice means, shared by the renderer and by anything drawing over it.
- `kernel-2d/runtime/scene/scene-view.ts` — the second surface of P2 as amended: one game, told which scene and which textures, reporting what it drew.
- `kernel-2d/runtime/scene/entity-layer.ts` — display objects kept in step with a list of entities by id, updated in place rather than rebuilt, and the `getBounds()` report of G8.
- `kernel-2d/runtime/scene/coordinates.ts` — the y-up-to-y-down flip, the rotation sign, the camera, and the pixel-grid snap of P5 — with no Phaser import, so every one of them is testable without a browser.
