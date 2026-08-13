---
name: editor-kernel
description: The architecture constitution for AI-built game editors — the core decisions, invariants, and contracts that every editor kernel must obey (document model, undo/redo, tool dispatch, persistence boundaries). Consult this skill whenever building or modifying the kernel of a game editor, deciding how editor state is structured, or resolving an architectural question about how the editor's pieces fit together.
---

# editor-kernel

The architecture constitution for the game-editor kernel. This skill holds the load-bearing decisions about how an editor is structured — the document model, how state mutates and undoes, how tools dispatch, and where the boundaries sit between editor, runtime, and persistence. It is the highest-authority skill in the library: when another skill conflicts with a decision recorded here, this one wins.

**Provenance of the current content.** The entries below were seeded from the feasibility report (`kernel-2d/docs/ai-game-tooling-report.md`, §4/§5/§8/§9) before the first kernel session, so the kernel would be built against a written constitution rather than one reconstructed after the fact. They are *design intent*, not yet *earned experience* — each is marked `[seeded]` and dated. As sessions hit reality, entries get confirmed, amended, or contradicted; session-earned content marked `[earned]` supersedes seeded content on any conflict, because a decision that survived contact with running code outranks one that survived only a document.

---

## Decisions

### D1: Three layers — runtime, editor, data — and the runtime contains zero editor code

The runtime ships with the game (render loop, entity layer, input, gameplay code). The editor never ships. The data is the game itself as text. **Reason:** the exported game is runtime + data + assets and nothing else; any editor code that leaks into the runtime becomes dead weight in every shipped build, and the separation is only cheap to maintain if it is absolute from the first session. _[seeded 2026-08-11, report §4]_

### D2: The editor embeds the actual runtime as its viewport

The editor imports the runtime as a library and instantiates it in the center panel. Play mode destroys the edit scene and boots the runtime's real scene loader against the same JSON. **Reason:** eliminates "looks different in the editor" divergence by construction — there is only one renderer, so the preview cannot lie. This is the Godot/Unity architecture. _[seeded 2026-08-11, report §4]_

**Confirmed in build, and sharpened by what "cannot lie" turned out to mean.** The first thing the runtime drew was one texture with its import settings applied. Three things settled by it:

1. **The line is about the renderer, not about the canvas.** The temptation is to put everything the viewport shows into `runtime/`, because it is all "the preview". But a frame guide and a pivot marker are annotations *about* a texture — no shipped game draws either — so they are editor chrome, and editor chrome inside the shipping layer is exactly what D1 forbids. They went into an SVG overlay in `editor/`, which also keeps a line one pixel wide at any zoom and costs the renderer nothing.
2. **What stops the overlay lying is a shared function, not a shared canvas.** The dishonesty worth fearing was never "these lines were drawn by different code", it was "these lines describe frames the runtime would not cut". That is fixed by exporting one definition of what a slice means from `runtime/` and having both the real slicing and the guides call it. Once the geometry is shared, drawing the guides in the renderer buys nothing and costs the boundary.
3. **The renderer reports what it drew; nothing downstream recomputes it.** `show()` answers with the placement it actually used and the frames it actually cut, and the overlay draws *those*. The alternative — the panel deriving the same placement a second time from the same inputs — is the U5/U9 failure one more layer out: two derivations that agree until they don't, with nothing on screen saying which one you are looking at.

_[earned 2026-08-11]_

**The second sentence, spent — and what it turned out to be worth.** Play mode arrived: the runtime's own loader (`runtime/scene/load-scene.ts`) opens a level file, follows its prefab and texture references, reads each `.meta` for itself, and hands the renderer a request. Pressing Play swaps which half of the editor produced that request and changes nothing else — same renderer, same canvas, same camera, so Stop has nothing to tear down.

Three things it cost, none of them visible from the decision as written:

1. **"There is only one renderer, so the preview cannot lie" is a weaker claim than it sounds, and it stops being true the moment play mode exists.** One renderer means the *drawing* cannot diverge. It says nothing about the code that decides **what to draw**, and there are now two of those: the editor's resolution of a level, and the runtime's. That is D25's failure one layer out, and it is invisible — both halves produce something that looks like a level. So the agreement is **checked rather than argued for**: the editing view's own report is kept at the instant Play is pressed and compared with the running level's, entity by entity, in the units the human sees ("Knight is drawn 4px left of where the editor drew it"). A divergence is named under the canvas and fails the suite. See `editor-verification` V24.
2. **The two loaders are deliberately not merged**, and the reason needs saying so a later session does not "fix" it. The editor's resolution is incremental, reactive and per-file, because a prefab edited in the Inspector has to land in the picture with nothing told to refresh (D25). The runtime's is a one-shot batch read with no store, no folder tree and no subscriptions, because that is what a shipped game has. Forcing either through the other breaks the thing that makes it right. The comparison is what keeps them honest, and it is far cheaper than the abstraction that would have unified them.
3. **A subject compared by value needs the derivation in the key as soon as there are two derivations.** The editor compares the renderer's request by `JSON.stringify`, so that a neighbouring file changing does not redraw. When the two halves *agree* — the intended outcome — the two requests stringify identically, so pressing Play compared by value, found no change, and drew nothing: the editing picture stayed on screen wearing play mode's caption. **The most misleading failure this feature could have, and it appears exactly when everything else is working.** The fix is one string prefix (`editing` / `playing`); the lesson is general.

_[earned 2026-08-12]_

### D26: What the runtime cannot do for itself arrives as an argument, and the editor's version is an adapter

The scene loader takes a `ProjectReader`: read one JSON file by project-relative path, and answer with a token that changes when an asset's bytes do. The editor answers out of the development service (`kernel-2d/editor/shell/project-reader.ts`); a shipped game answers with `fetch` and a build number.

**Reason:** D1 says the runtime contains no editor code, and the obvious reading is "do not import the editor". The reading that actually bites is **do not import the editor's *situation*** — a service on localhost, a folder tree held in React state, an endpoint that addresses import settings by the path of the file they annotate. A loader that knew any of that would be untestable outside a browser and unusable outside the editor while importing nothing forbidden. Two functions is the whole seam.

Four things learned building it:

1. **The adapter is allowed to be an adapter.** The loader asks for `knight.png.meta`, because that is the file on disk and that is what an export will fetch; the service addresses it as `knight.png`, so the editor's reader takes the suffix off. Pushing that quirk into the loader "so both sides agree" would put a development service's URL scheme inside the shipping layer.
2. **A seam moves some tests somewhere the product cannot reach.** In the editor the service parses documents before the runtime sees them, so the runtime's *own* validation never runs in the browser and a level broken by hand is described by the service's sentence rather than the loader's. That path is only real in an export — so it is covered by unit tests that hand the loader garbage directly. Ask it of every injected seam: which side of this does the running product actually exercise?
3. **The version token must come from the same place as everybody else's.** It is half the renderer's texture cache key, so a loader that invented its own would upload one file to the graphics card twice and show a texture from before it was re-saved. Shared cache key, not a private detail.
4. **Fatal is only the document you were asked for.** A missing prefab, a texture with no import settings beside it, a reference pointing at a file that is no longer the one it was written against — all named, none fatal, and the level still runs. Not leniency for its own sake: the editor's own resolution already fails that way, and the two pictures are only comparable if both halves fail identically. _[earned 2026-08-12]_

**Both halves now exist, and there are three readers rather than two.** The shipped game's is `fetch` plus a constant build number (`kernel-2d/runtime/web/start-game.ts`); the export command's is `readFile` plus a constant (`kernel-2d/scripts/export/project-reader.ts`), so it can validate a project with the loader the game will use rather than a second opinion about what a level reaches. Three things the far side of the seam settled:

1. **The plain implementation is the shipped one, and the editor's is the special case.** Point 1 above said "the adapter is allowed to be an adapter"; with all three written, the shape is clearer than that — the two constant-version readers are four lines each and do no adapting at all, and every quirk lives in the development service's. That is the right way round, and it is worth checking on any seam: if the *shipping* implementation is the complicated one, the seam is in the wrong place.
2. **The version token being a constant is a decision, not a stub.** Nothing changes underneath a build, so one build is one number and every texture reaches the graphics card once. Do not reach for a content hash or a cache-buster: it would make one build's URLs differ from the next's, which breaks "export twice, get the same folder" for no reader's benefit.
3. **A seam whose consumers span both TypeScript projects constrains what the module may import.** See D16's third paragraph — this is where that bit.

_[earned 2026-08-12]_

### D13: Export targets are kernel-level, one command each

`npm run export:web` (Vite static build) and `npm run export:desktop` (native shell). **Reason:** the runtime is web-native, so both targets are a packaging step over identical game code — which makes the desktop-shell choice deferrable and reversible. Building both into the kernel means every genre editor inherits them instead of re-deciding. _[seeded 2026-08-11, report §4]_

**The web half is discharged, as `npm run export -- <project> [--out <folder>]`, and it turned out to be four decisions rather than a packaging step.** `kernel-2d/scripts/export.ts` plus `scripts/export/`. Each of these had an obvious wrong answer and the wrong answers are all cheap-looking.

1. **Deciding and writing are separate functions, and every refusal lives in the deciding one.** `planExport` reads the project settings, runs the runtime's own loader, checks the things the loader does not, and answers with either a file list or one sentence. `writeExport` takes that and touches the disk. Two payoffs, and the second is the one that matters: a refused export cannot leave a half-written folder because nothing had been written yet, and **a refusal is testable without a bundler** — which is why there are ten of them rather than the two somebody would have written if each cost a four-second build. Ask of any command that produces an artefact: can its refusals be tested without producing one?

2. **What goes in the folder is the transitive closure of the startup scene, not the project folder.** The starting level, the prefabs it places from, the textures those draw, each texture's `.meta`, and `project.json`. Not `assets/` — which would ship `assets/source`, the `.psd` originals, and the art that was cut from the level last week. Two things make this safe rather than clever: the walk is `loadScene`'s, so it is the same resolution the game will do rather than a second opinion about what a level needs; and **the command prints what it left out, counted by folder**, because "only what the game needs" has to be checkable by the human rather than taken on trust. Today the closure is *complete* rather than merely sufficient — nothing can reach a second level yet — so the export grows with the game rather than with the folder.

3. **An export refuses where the editor merely reports, and that asymmetry is the design.** A missing prefab, a texture with no `.meta`, a texture whose bytes are gone: in the editor each is named under the viewport and the rest of the level runs, which is right because the human is mid-work. In an export the same problem stops being information and becomes a shipped defect, and the command is the last place the person who can fix it will see it. So: anything meaning *something would not be drawn* refuses, before a folder exists. The exception is D5's witness-only pair (`prefab-different-file`, `texture-different-file`) — those draw, the id deliberately does not veto, and swapping a file is something a human does on purpose, so they warn and carry on. **One `LoadProblem` union, two policies, and the policy lives with the caller** rather than in the loader.

4. **The privilege question again, and it is the same question a fourth time.** D17 and D22 kept asking "what does this do if the caller is confused?"; an export folder asks it about a folder rather than a file, and the answer is a manifest of what the last export wrote, kept in the folder. A folder holding a file the manifest does not list is refused **before the build**; a folder with anything in it and no manifest is refused outright; anything the manifest lists that the game no longer needs is deleted, which is what makes exporting twice give the same folder rather than an accumulating one. The mistyped `--out` is the case this exists for.

Two more that are specific and would be re-derived painfully. **The export refuses to write inside the project folder** — the watcher would create a `.meta` beside every copied PNG, so the folder would be modified behind the human's back *and* two exports would stop being identical. And **it verifies its own output** (`editor-verification` W15): the import boundary is checked on the source and this is checked on the artefact, and the two fail apart. _[earned 2026-08-12]_

### D20: A module the runtime needs lives in `runtime/`, whoever else uses it

The `.meta` format was written for the sidecar and lived there. The moment the runtime had to read import settings it had to move, because the sidecar is development-only and `runtime/` importing it would have broken D1 in the first file of the new layer. It now sits at `kernel-2d/runtime/formats/meta-schema.ts`, and the sidecar and editor both import it from there.

**Reason:** D16 already fixes the *direction* — the shared module owns the vocabulary and the Node module imports it — but says nothing about *where* the shared module should sit, and "beside whoever wrote it first" is the answer that quietly fails. Of the three layers only one ships, so the rule that keeps working is: **anything the shipping layer reads belongs to the shipping layer.** Ask it when a format is created, not when the runtime finally needs it; the move is mechanical either way, but it is a dozen import lines now and a merge conflict across a session's work later.

Two things learned doing it, both worth having in advance:

- **A module compiled by both TypeScript projects may have relative imports, and they are written `./x.js`.** ~~It must have none of its own.~~ That was recorded as a constraint and it is false — measured 2026-08-12 by trying it. TypeScript maps `./x.js` → `x.ts` under bundler resolution as well as NodeNext, and Vite does the same at runtime, so one spelling satisfies both projects. The *extensionless* spelling is the one that only works on one side, so a shared module must not use it. See `text-formats` T14 for what believing the stronger version cost.
- **The boundary is worth a test, because the failure is silent.** `kernel-2d/tests/architecture/boundaries.test.ts` reads every file under `runtime/` and fails if any of them imports from `editor/`, `sidecar/`, `scripts/` or `tests/`, and separately holds the runtime's external dependencies to a named list. An import in the wrong direction compiles, passes every behaviour test, runs perfectly in the editor, and surfaces as a shipped game carrying a React panel. There is no moment at which it announces itself.

**Fired a third time, and the third one is the useful evidence: the thing that moved was a piece of UI *policy*.** The zoom ladder — the list of whole-number magnifications a picture may be drawn at — lived in `editor/shell/zoom.ts` and was explicitly labelled editor policy, in a comment inside `coordinates.ts` saying so. Then the exported game had to frame its own starting level, which means picking a scale, and it wanted the *same* rule for the *same* reason: pixel art at 3.4× has some rows two pixels tall and some three. So `ZOOM_STEPS` and `fitStep` moved to `runtime/scene/scale-steps.ts` and the editor imports them; `stepUp`, `stepDown` and `describeZoom` stayed behind, because buttons and the wording `8×` are genuinely the editor's and a shipped game has neither.

**The refinement to the rule:** "which layers read this?" is the right question, and *"is this policy or is this data?"* is not — a policy the shipping layer has to enact belongs to the shipping layer just as much as a format does. The tell is a comment asserting that something is one layer's business; when the other layer needs it, that comment is the thing to re-read rather than to work around.

_[earned 2026-08-11, third case 2026-08-12]_

### D21: The service serves the bytes of an asset, and only of an asset

`GET /asset?path=…` hands over one file, and the rule is four lines: only a file the editor recognises as an asset (the extension vocabulary in the `.meta` schema — textures, audio, fonts, never a `.meta`, never a scene); never outside the project folder, through the same path check every other browser-supplied path already passes; a read that creates, writes and lists nothing; and a content type from the extension, never guessed from the contents.

**Reason:** the same discipline as D17's three lines, applied to reading. The obvious implementation is a static file server rooted at the project folder, and it is a much larger privilege than the feature asked for — after it exists nobody can say what the editor is able to read without going and looking. Narrowing it to the asset vocabulary costs one line and means the answer is already written down somewhere that has to stay current, because it is the same list that decides which files get a `.meta`.

Two practicalities: stream the file rather than reading it, or the size of somebody's art becomes the size of this service's heap; and answer `no-store`, because the file on disk is the record and the editor is expected to show a re-saved texture within the second. _[earned 2026-08-11]_

### D23: The viewport's camera is a scale and an offset in the coordinate module, never the engine's own camera

`runtime/scene/coordinates.ts` gains `Camera = { scale, focus }` and the arithmetic around it — pan, zoom-about-a-point, frame, and the inverse of each. Phaser's `setScroll`/`setZoom` were considered and declined.

**Reason: the renderer reports what it drew in screen pixels, and the editor's overlay draws those numbers (D2).** An engine camera moves the authority over where things land into the engine, and `getBounds()` then answers in world coordinates — so either the overlay converts them, which is the second derivation this arrangement exists to prevent, or the renderer converts them back, which is this arithmetic anyway with the engine's underneath it. Keeping it here also keeps the module free of any engine import, so panning and zooming are testable with no browser, no canvas and no renderer. The cost, paid knowingly: about twenty lines of arithmetic, and no free ride on whatever the engine's camera does next.

Three things settled by building it, none of them visible from the outside:

1. **The offset is the scene point at the *centre* of the canvas, not at a corner.** Identical information, and it is the difference between a panel dragged wider revealing scene on both sides and one that shifts everything already on screen. "I resize the panel and do not lose my place" then falls out of the shape of the data rather than being arranged for.
2. **Snap the camera onto the device pixel grid once; never snap each sprite as it is drawn.** Both keep pixel art crisp. Only the first keeps the distance between two entities exact at every zoom — with per-sprite rounding, the measured extent of a level comes out slightly different depending on how far in you happen to be, so **framing stops being idempotent**. That surfaces as pressing the frame key twice and getting two different zooms, which reads as a broken key rather than as a rounding decision three files away.
3. **Framing is a one-shot press, not a mode.** A texture preview is something you look at, so it can sensibly stay in a fitting mode; a scene camera is something you drive, and a resize has to keep your place rather than reframe. The two viewports share a zoom ladder and deliberately do not share this.

4. **The crossing into screen space changes the sign of a rotation *and* its unit.** A level stores rotation in **degrees**, because that is what a designer types into a field; the renderer wants **radians**, because that is what the engine takes. The flip of sign — y is up in a scene and down on screen, so the same visual turn is the opposite angle — is the half that gets remembered, and it is the half that is visible when you get it wrong. Getting the *unit* wrong is a factor of about 57 and looks, on screen, like a sprite that simply refuses to point anywhere sensible. Both conversions live in the one function that crosses, and the name says which way it goes. _(Added 2026-08-12: the first parity drill regenerated this module with the sign and not the unit, which is exactly the half the earlier wording carried.)_

_[earned 2026-08-12]_

### D3: All authored state is human-readable text in the project folder

No binary project database, no hidden index of record. **Reason:** three compounding payoffs — git-diffable history, and the ability for a session to inspect live game state with `grep` instead of loading it through the editor. The third is context economics: a well-factored text project lets a session read the schema rather than the codebase, which is what keeps sessions cheap as the kernel grows. _[seeded 2026-08-11, report §3/§4]_

### D16: Modules compiled into the browser may not reach for anything Node-only, even for a type

The schemas are shared between the sidecar and the editor. A shared module that type-imports a Node-facing one drags the whole graph into the browser project, and `tsc` reports it as a pile of "cannot find module 'node:path'" errors in files nobody touched. The fix is always the same and always in the same direction: the **shared module owns the vocabulary, and the Node module imports it** — not the reverse. Display helpers used by both the terminal and a panel move into their own dependency-free module rather than being duplicated. **Reason:** the two TypeScript projects (`editor-ui` U4) are what enforce this, and they only enforce it if the boundary is respected in the import direction; a shared module that quietly depends on Node is a browser build waiting to break. _[earned 2026-08-11]_

**The full statement, learned when a shipping module had to be compiled by the Node project as well: a module the Node half must compile may reach only modules the Node half can compile, and its relative imports carry the `.js` spelling.**

The export command validates a project by calling `runtime/scene/load-scene.ts` in plain Node. That one import failed twice over, and neither failure named the cause. The loader's own relative imports were extensionless, which NodeNext cannot resolve; and it took two *type-only* interfaces from `runtime/scene/scene-view.ts`, which imports Phaser and uses `document` — and a type-only import is still **resolved and type-checked**, so the Node project reported a pile of missing-DOM errors inside a renderer nobody had touched. The fix is mechanical and worth reaching for immediately: move the shared interfaces into a module with no engine and no DOM in it (`runtime/scene/scene-request.ts`), re-export them from the renderer so nothing downstream notices, and give the loader `./x.js` imports (`text-formats` T14 — that spelling satisfies both projects).

**Reason, and the rule to apply going forward:** `runtime/` is compiled by the browser project because it ships into a web page, and that is right — but the moment a development-only process (a build command, a migration script, a Node test) has to *call* something in there, that module joins a second compiler with a smaller `lib` and stricter resolution. Ask it when a runtime module gains a Node caller, not when the errors arrive: **which of my imports can Node compile?** The three tells are an extensionless relative import, a type-only import from a module that touches the DOM or the engine, and an import of the layer's barrel. `import type` erasing at runtime is the trap — it buys you nothing at compile time.

**The narrower mirror image, learned earlier when a module wanted a unit test: a module you want testable outside a browser must not import the layer's barrel.** `runtime/index.ts` re-exports the renderers, so *any* import from it loads Phaser — which needs a `document`, and which turns a pure-arithmetic module into one that cannot be tested without booting a browser. The fix is to import the modules themselves; type-only imports are erased entirely, so a file can take `ShownScene` from the renderer's own module and pull in nothing at all. **Reason:** the barrel is a convenience for the application and a trap for everything else, and the failure is a stack trace from inside a vendored bundle that names no file you wrote. Rule of thumb: application code may use the barrel, and anything with a unit test beside it may not. _[earned 2026-08-12]_

### D4: Sidecar `.meta` files; the asset browser mirrors the folder 1:1

The binary lands wherever the human put it from Photoshop or Blender. A JSON sidecar beside it (`knight.png` + `knight.png.meta`) holds import settings: slicing, pivot, filtering, ~~collider generation~~. No import step, no copy, no rename.

**Struck 2026-08-12 by the first parity drill: there is no collider generation and there never has been.** A texture's settings are filtering, pivot and slicing — three, not four. The fourth was seeded from the report as design intent and has read as current ever since, which is how a session comes to regenerate a field nothing writes and nothing reads. Colliders may well arrive; when they do this is the line to un-strike. The general form is worth more than the detail: **a seeded entry listing what something contains is a claim that ages, and nothing about it announces the day it stops being true.** **Reason:** the folder *is* the database. The editor's only privilege over the human's files is annotation — the moment it starts moving or rewriting them, the 1:1 guarantee dies and the human loses the ability to work in their own tools without the editor's permission. _[seeded 2026-08-11, report §4/§8]_

**Confirmed in build: the mirror includes the sidecars.** The asset browser lists `.meta` files as the ordinary files they are. It is tempting to hide them for tidiness, and it is wrong: a browser that silently omits part of the folder is no longer a mirror, and the human then has files on disk the editor has taught them not to expect. Folding a sidecar into the row of the asset it annotates is a presentation job for the feature that gives those files meaning — not a reason to drop them from the tree. The ignore list stays what it was: tooling junk only, never anything a human might have authored. _[earned 2026-08-11]_

**Amended, once that feature existed: sidecars fold into the row of the file they annotate.** `knight.png` is one row carrying a marker rather than two rows; selecting it shows the settings in the inspector. This is the presentation job the note above anticipated, not the hiding it forbids — every file on disk is still represented, and the folder counts the inspector shows are computed from the same rule that lists the rows, so the number and the tree can never be two different answers. **A `.meta` with no file beside it keeps its own row, marked**, which is the same rule rather than an exception to it: with nothing to attach to, the only honest place to show it is on its own, and a stranded sidecar is exactly the thing a human needs to be able to find. Put the folding rule in one shared module the moment a second panel needs it; two copies disagree the first time either changes. _[earned 2026-08-11]_

### D17: The sidecar creates a `.meta` that is missing, deletes a stranded one at startup, and replaces one the editor names — nothing else, ever

Stated in full, because this is the whole of what the service may write into a human's project folder:

- **Of its own accord**, creates a `.meta` when a file with a recognised asset extension has none — at startup, when a file is added, and when a sidecar is deleted while its file is still there — and deletes a stranded one at startup, only at startup.
- **Replaces** the entire contents of one `.meta`, and only when the editor names that file and supplies a complete, valid document.
- **Never modifies anything else**, and never changes a `.meta` on its own initiative, including one it cannot parse.

**The middle line arrived later than the other two, and the third is what stops it meaning more than it says.** The editor gaining the ability to write a file is not the service gaining the ability to write files: nothing in the service decides on its own that a `.meta` should be different from how it found it. Every widening of this list has to be phrased so that it still fits in three lines — if it does not, the privilege has grown past what anyone can hold in their head, which is the point at which it starts damaging work nobody authored.

Three refusals belong to the write, each leaving the file byte-for-byte as it was: a path that is not inside the project, a document that is not a valid `.meta`, and a sidecar with no file beside it (writing that one strands it, and a stranded sidecar is deleted at the next startup — so the change would quietly not survive the day). What it deliberately does *not* refuse is a document whose type disagrees with the file's extension: what a file says it is beats what its name suggests (`editor-ui` U11), so that is an authoring decision rather than a mistake to correct.

**Reason:** the write privilege has to be small enough to state in three lines, because every widening of it is a way for the editor to damage work it did not author, and the damage is discovered later by a human who has no reason to suspect the tool. Three specifics that carry their own reasons:

1. **Create-when-missing uses an exclusive-create flag, not an existence check followed by a write.** The check-then-write version has a window; two sidecars opened on the same folder, or a startup sweep racing a live event, land in it. `EEXIST` is not an error here, it is the answer.
2. **A file it cannot parse is left alone.** A broken `.meta` is far more likely to be one a human is midway through editing than one worth replacing, and "the editor overwrote my file because it did not like it" is unrecoverable.
3. **Deleting strays happens at startup only.** No OS reports a rename as a rename (G7), so a stranded sidecar during a session is as likely to be the removal half of a rename as it is to be rubbish — deleting there would throw away the stable id and settings of every file the human renames, mid-gesture. Doing it once at startup keeps that window shut. The cost, which was accepted knowingly rather than discovered: a rename *does* lose its id and settings at the next start, and that is the material a "fixup when files move" tool would otherwise have used. **Every removal is named in the startup banner**, because this is the one moment the service deletes anything and silence there would make the loss invisible. _[earned 2026-08-11]_

**The rule is unchanged and now covers much less ground, which is the more useful thing to know.** Once the editor could rename a file itself (D22's fifth and sixth lines), a rename it performed stopped being two events that have to be correlated: it is one request that moves the file and its sidecar together, so it never produces an orphan and never loses an id. This rule is not what saves that case. What is left under it is renames made *outside* the editor, where the two events genuinely are all the operating system offers.

The general shape, which is worth more than the detail: **a rule written to survive an ambiguity stays correct when the ambiguity is removed, and starts costing more than it buys.** The cost here — a file moved in Explorer still loses its settings at the next start — is now paid only by somebody working outside the tool, which is a much easier trade to defend than the original. Re-read a rule of the form "we cannot tell X from Y" whenever something new makes X knowable; the rule usually does not need changing, but its *scope* does, and nothing announces the day it shrank. _[earned 2026-08-12]_

### D22: The write privilege now covers documents, and the two guards that keep it from meaning "write anywhere"

D17 restated, with the middle line widened so the editor can put a scene back:

- of its own accord, creates a `.meta` when an asset has none, and deletes a stranded one at startup;
- **replaces the whole contents of one file the editor names, when that file already exists and the document is valid in a format this editor knows;**
- never modifies anything else, and never changes a file on its own initiative.

Still three lines, and it stays three lines as prefabs and data tables arrive — that was the point of phrasing it around "a format this editor knows" rather than naming scenes. The registry of known formats is then the thing that has to stay current, and it is one object in one file.

**"Valid document" is a statement about the body. Two more are needed about the target, and both are load-bearing:**

1. **It never creates a file.** A path with nothing at it is refused rather than filled in. Making a new scene is a feature that does not exist yet; when it does it should be a deliberate one rather than a side effect of a typo in a path.
2. **It only replaces a document with one of the format that is already there**, which means reading and parsing the existing file before writing. Without this, a perfectly valid scene document sent at the path of somebody's PNG overwrites their art — the document is valid, the path is inside the project, and *every other check passes*. This is the failure to hold in mind when widening any write: the checks were all about the thing being written and none of them about the thing being written over.

**Widened once more, to four lines, when levels had to be startable — and the shape of the widening is the lesson.** Making a file is a *separate request* (`POST /document`) from replacing one (`PUT /document`), not a flag and not a fall-through:

- of its own accord, creates a `.meta` when an asset has none, and deletes a stranded one at startup;
- **creates one file the editor names, when there is nothing at that path and the document is valid in a format this editor knows;**
- replaces the whole contents of one file the editor names, when that file already exists and the document is valid and of the format already there;
- never modifies anything else, and never changes a file on its own initiative.

**Reason:** the obvious design is one write that creates when there is nothing there, and it means a single mistake in the editor turns "make a new level" into "erase this level". Kept apart, the create refuses when anything is at the path and the replace refuses when nothing is, so a confused caller gets a sentence instead of a destroyed afternoon. That generalises into the question to ask of every widening of this file: **not "is this operation safe?" but "what does this operation do if the caller is confused about which one it wanted?"** Both halves get tested from both sides, or only the half somebody remembered is real (V13).

Two guards belong to the create specifically. **It never makes a folder** — a path whose parent is not already there is refused, because creating folders is a second privilege wearing the first one's clothes and a mistyped path would quietly grow a tree of them. And **it is exclusive at the filesystem** (`wx`), not an existence check followed by a write: the check-then-write version has a window that two editors on one folder land in, and `EEXIST` is not an error there, it is the answer. _[earned 2026-08-12]_

**Keyed on the document's own `format` string, never on the path.** Where a file sits in the folder tree is a convention (`scenes/` is in the folder map, not in the code); what a document says it is, is a fact — the same ordering as `editor-ui` U11, and the first real payoff of the format literal every document has carried since T1. A consequence worth wanting: "this is not a format I know" and "this is a format I know and the file is malformed" are then *different* answers with different sentences, where a single discriminated union over all known formats would have collapsed them into one unhelpful "invalid". _[earned 2026-08-11]_

**Widened to six lines by renaming, moving and deleting — and the useful part is the invariant that made six defensible, not the two new lines.**

- of its own accord, creates a `.meta` when an asset has none, and deletes a stranded one at startup;
- creates one file the editor names, when there is nothing at that path and the document is valid in a format this editor knows;
- replaces the whole contents of one file the editor names, when that file already exists and the document is valid and of the format already there;
- **moves one file or folder the editor names to one path the editor names, when there is nothing at the destination — taking a `.meta` with the file it annotates;**
- **deletes one file the editor names, and the `.meta` beside it;**
- never modifies anything else, and never changes a file on its own initiative.

The rule "every widening must still fit in a few lines" was going to run out, and counting lines was never really the test. What the list is actually for is being *holdable*, and six lines are holdable because every one of them has the same shape: **the editor names a path and one thing happens to it; the service never picks a path itself, and never touches a second file except the `.meta` belonging to the first.** That sentence is what a seventh line has to fit inside — and it is what rules out the widenings that would genuinely be dangerous, like a service that decides for itself which files a rename should repair. **Replace a counting rule with the invariant it was standing in for, at the point where the count stops meaning anything.**

Five things specific to these two verbs, each of which had an obvious wrong answer:

1. **A move renames the `.meta` first and the file second; a delete removes the file first and the `.meta` second.** Opposite, and both chosen against the same hazard — this service watches the folder it writes into, and its own create-when-missing rule is listening (G9's loop, arriving from a new direction). Move the file first and the `added` event can reach `ensureMetaFor` before the sidecar lands: a **fresh id is minted and the pivot and slicing are gone**, which is precisely what the feature promised to preserve. Settings-first has the harmless failure instead — a stray sidecar at the old path, which the startup sweep already clears. Delete the `.meta` first and the "a sidecar was deleted while its file is still there" rule writes new settings for a file that is about to vanish; file-first has no race at all. **Order the two writes so that losing the race costs the recoverable thing.**
2. **The window is closed by a rule, not by timing.** Every path an operation is about is held in a set for the length of the request, and the watcher handler skips those: *the service does not act on its own file operations*. Same instinct as G9 and G10, said once rather than left to how fast chokidar happens to be.
3. **This is the one check-then-act in the service, and it should be admitted rather than hidden.** Everywhere else "is that path free?" is asked of the filesystem in the operation that takes it (`wx`). **Node has no no-overwrite rename** — `fs.rename` replaces silently on both platforms and `RENAME_NOREPLACE` is not exposed — so the destination is checked and then used, with a window between. Say so where the code is, bound the damage, and do not paper it over with a second check that only makes the window shorter.
4. **Delete is one file, never a folder.** "Deletes one file the editor names" fits in the list above; "deletes a folder and everything under it" is a blast radius, and the question the editor asks before deleting — what still uses this? — stops having a short answer when it is about a hundred files. Rename and move take folders happily, because they destroy nothing.
5. **Two more paths worth refusing, both about the `.meta` convention rather than about the filesystem.** A sidecar named on its own is refused (it would strand or silently detach a file from its id), and so is renaming a file *to* something ending `.meta` (it becomes a sidecar for a file that does not exist, and the startup sweep deletes it — the change would quietly not survive the day). Whenever a naming convention is load-bearing, both directions of it are.

_[earned 2026-08-12]_

**Tested by a second format arriving, and it held: prefabs cost one line in the registry and widened nothing.** Same create, same replace, same refusals, same four lines. Worth recording as a *result* rather than as an intention, because "it will generalise" is a claim every design makes and few keep.

One guard only becomes load-bearing at the second format, and it was already there: **a document is only ever replaced by one of the format already at that path.** With one format that reads as pedantry. With two it is the line between "rename this prefab" and "a valid prefab written over somebody's level" — a document that is valid, at a path inside the project, passing every check that is about the thing being *written*. Assert it in both directions the day a second format lands; the direction nobody thought of is the one that breaks. _[earned 2026-08-12]_

### D24: A document is its own annotation, so it carries its own id — no sidecar

A prefab holds `id` as a field of the document. A texture's id lives in the `.meta` beside it. Both are D5 references from the outside and neither reader can tell the difference, which is the point.

**Reason:** the `.meta` exists for one reason — nothing can be written inside a PNG. Reaching for one anyway when the file *is* JSON would buy nothing and cost the whole of D4's orphan machinery: a second file to keep beside the first, a stranded-sidecar rule, a startup sweep, and a `.meta` for a `.meta`'s worth of confusion. Ask "could this file hold the annotation itself?" before adding a sidecar for anything.

Three consequences that are easy to get wrong in the other direction:

1. **Nothing gives a document a `.meta`.** The extension vocabulary that decides which files get one (D21) already excludes `.json`, so this falls out — but it is the reason the Inspector describes a `.json` by what is *inside* it rather than by what is missing beside it.
2. **Setting a reference to a document is synchronous**, where setting one to a texture is not (D5's cost note). The id is in the document the editor has already read, so there is no round trip. Do not copy the asynchronous shape over out of symmetry.
3. **The witness half still applies.** The id recorded in a level is compared with the id the prefab carries, disagreement is said out loud, and the instance is drawn anyway. Same three rules, different place to read the id from. _[earned 2026-08-12]_

### D25: A reference between documents is resolved by the format, once, and the resolved copy is never what gets written

An entity that is an instance of a prefab carries a reference and nothing else. `resolveEntity(entity, prefab)` — in `runtime/formats/scene-schema.ts`, not in the editor — merges the prefab's components under the entity's own, leaves the transform untouched, and hands back a copy.

**Three decisions, each of which had an obvious wrong answer.**

**Resolution lives in the format, not in the panel that first needed it.** The runtime's scene loader will do this same sum when play mode arrives, and two derivations of *what a level contains* is the editor and the shipped game disagreeing about the game — D2's failure with a longer fuse. Same instinct as D20: the shipping layer owns anything the shipping layer reads.

**"Editing the prefab updates every instance" is not a feature; it is what you get for free if the resolver reads the document store.** The prefab the Inspector edits and the prefab an instance draws are then one object, so a change lands in the picture with nothing told to refresh, nothing invalidated, and no cache. Reading the *served* answer instead would work exactly once, at open, and then quietly stop — and it would look correct in every test that only opens a level.

**The resolved copy is for drawing and describing, and writing one back would silently sever the link.** It carries the prefab's components; saved, they become the level's own, and every instance stops following the prefab with nothing on screen saying so. What keeps this true is not care, it is that every writer in the kernel re-finds its entity by id *inside* the transaction (D7) and therefore only ever touches the document. Say so where the resolved value is produced, and assert it by reading the file for what it does **not** contain (`editor-verification` V22).

A fourth, smaller: **what the entity carries itself wins, per component type.** The editor offers no way to write an override, but a hand-edited file has to mean something, and "the one written here beats the one it inherits" is both the least surprising reading and the shape overrides will need — so nothing has to change format to get them later. _[earned 2026-08-12]_

### D5: References carry both a stable ID and a human-readable path

`"sprite": {"id": "a3f9", "path": "assets/textures/knight.png"}`. IDs are generated once and stored; a fixup tool reconciles the pair when files move. **Reason:** the two properties are irreconcilable in a single field. IDs survive renames but make files unreadable; paths stay greppable and let a session understand a scene by reading it, but break on every move. Carrying both costs a few bytes and removes the tradeoff. _[seeded 2026-08-11, report §4]_

**Honoured for the first time by the scene format, and the division of labour is the part that was never written down.** Carrying both fields is easy; deciding what each one *does* is the decision, and the answer that works before a fixup tool exists is three rules:

- **The path resolves the reference.** It is looked up in the project folder, and that is the whole of how a sprite is found. This is what keeps a scene greppable and what lets a session understand a level by reading it.
- **The id is the witness.** Once the file is found, the id in its `.meta` is compared with the one the reference recorded. Disagreement means the reference points at a *different file* than the one it was written against — a texture swapped, a file restored from a backup — and it is said out loud.
- **The id does not veto.** A mismatch is reported and the sprite is drawn anyway, because the file at that path *is* what the scene points at, and refusing to draw it would be less informative than drawing it and saying so.

Without the third rule the id is a trap. Without the second it is dead weight — before this session nothing had ever read one, and an unread field is a field that is quietly wrong. There is also a cost to budget for: the id lives in the *texture's* `.meta`, so **setting** a reference is asynchronous (ask the service for the id, then write both halves in one transaction). Writing the path alone would be synchronous and would look identical on the day it was built. _[earned 2026-08-11]_

**The fixup tool this decision was written for now exists, and it is smaller than the decision implied — because the id does not need reconciling.** "A fixup tool reconciles the pair when files move" reads as though both halves drift and both get repaired. They do not. A file moved from inside the editor takes its `.meta` with it, so the id at the new path is the id the reference already recorded: **the repair rewrites `path` and never touches `id`.** That is the whole of it, and it is the payoff of carrying both fields rather than a cost of it — an id that changed on a move would make a rename a rewrite of every reference's *identity* rather than of its address, and there would be nothing left to witness with while it was happening.

Three things the tool settled that the decision could not have:

1. **Where the references are is a fact about the format, and belongs beside the component registry** — one map from component type to the field holding a reference, in `scene-schema.ts`, read by both "what does this point at" and "rewrite this". Derived by hand in the editor instead, it silently stops matching the registry the first time a component type is added, and the symptom is a reference that quietly stops being repaired.
2. **A file target and a folder target are one rule**: a reference matches when its path *is* the target or sits under it, and the rewrite swaps that prefix. Renaming a folder is then a hundred references for the same three lines — and the separator has to be in the test, or `assets/tex` matches `assets/textures`.
3. **A document that changed nothing must not be written.** The rewrite answers "nothing" rather than an equal copy, because writing every level in the project on every rename churns files the human never touched, and V22's "the neighbour that did not move" assertions are exactly about that. _[earned 2026-08-12]_

### D6: Every format is a Zod schema, and the schema file is the single source of truth

The runtime validates on load, the editor validates on save, and both read the same definition. Adding a component type changes the schema in exactly one place; `tsc` and the validators then reveal every tool that needs updating. **Reason:** this is the primary defense against serialization drift — the save format and load format quietly disagreeing — which is the most common failure mode of AI-built editors. See G1. _[seeded 2026-08-11, report §4]_

### D7: Undo is document-level, via immer patches through a transaction API — never per-tool

All mutations go through the kernel's transaction API. Undo is implemented once as patches over the in-memory document, not as per-tool inverse commands. **Reason:** per-tool undo is the #1 source of editor jank, because every new tool is a fresh chance to write an inverse operation that is subtly wrong, and the bug surfaces three actions later where nobody is looking. Document-level undo means every future genre tool inherits correct undo for free, having written no undo code at all. This decision is load-bearing enough that a session bypassing the transaction API is a defect regardless of whether its feature works. _[seeded 2026-08-11, report §5/§13]_

**Confirmed in build, with the shape it actually took.** `kernel-2d/editor/store/documents.ts`, built on zustand 5 + immer 11. Six things settled by writing it, each of which a fresh session would otherwise re-derive the hard way:

1. **The store is created inside the module and never leaves it.** What is exported is the transaction API plus a read-only view — `getState`/`getInitialState`/`subscribe`, deliberately not `setState`. Together with immer's auto-freeze, a panel that assigns to a document throws at the point of the assignment. That is what makes G2 structural rather than advisory: the failure is loud, immediate, and in the tool that caused it, instead of silent and three actions downstream.
2. **There are exactly two doors, and the second one is not an edit.** `edit`/`editDocument` records undo; `adoptFromDisk` does not, because the file changing underneath the editor is a reload rather than something the human did — offering to undo somebody's text-editor save is incoherent. Two is the right number; if a third ever appears, check whether it is really a reload in disguise.
3. **One stack for the whole project, ordered by time.** Per-file stacks make Ctrl-Z mean something different depending on what is selected, which is the jank the API exists to prevent.
4. **Patches are rooted at the document map, not at the whole store state.** The first path segment is then the document's own path, which is how the writer knows what to save — and it puts everything that is *not* undoable (what is believed to be on disk, what is in flight, what failed to save) structurally out of a patch's reach. What a recipe cannot see cannot be undone by accident.
5. **A transaction that produced no patches records no undo step.** A control re-emitting the value it already holds is not a change, and an undo entry that reverses nothing is a step Ctrl-Z appears to skip — G2's symptom, arrived at from the opposite direction.
6. **Coalescing merges inverse patches in reverse order.** The merged entry is `patches: [...old, ...new]` and `inversePatches: [...new, ...old]`, because undoing both means undoing the newer change first. The naive same-order version passes every test written against one field being replaced twice, and fails the first time a run touches more than one key — which a frame-size run does.

**The bet paid, and here is the evidence rather than the argument.** A second format arrived in the store — scenes, holding a list of entities — and **not one line of undo, merging, coalescing, staleness or save logic was written for it**. `Document` became a union, one branch on `format` chose which service endpoint the write goes to, and everything else was already true. Three things learned collecting that:

- **Adding and deleting are edits.** This is where a session reaches past the transaction API, because *creating* something feels categorically different from *editing* it. It is not: an add is a recipe that pushes onto a list, a delete is one that splices, a reorder is one that moves a slot, and Ctrl-Z covers all three for free. A tool that writes its own inverse for a create is a defect whether or not it works (G2).
- **A recipe re-finds its target by id rather than closing over an index.** Between the click and the recipe running, a text editor may have changed the file underneath — and an index into a list that has moved on deletes the wrong thing. The cost is a `findIndex`; the failure mode is silent and destructive.
- **What is selected after an edit is not part of the edit.** Selecting a newly added entity happens *outside* the transaction, or undo would restore a selection as well as a document, which is exactly what `editor-ui` U8 exists to prevent.

**Where the line falls, stated because the first thing outside it arrived: making a file is not an edit.** The store holds documents that are *open*, and a file that does not exist yet is not one of them — so creating one does not go through the transaction API, does not appear in the map until the watcher reports it, and **is not undoable**. That is the honest answer rather than a gap: Ctrl-Z reverses changes to documents and has never deleted a file, and teaching it to would make one key mean two very different sizes of thing. Say so in the hand-off; a human who assumes otherwise finds out at the worst moment. The tell for the next case: if the operation's inverse is "delete something", it does not belong on this stack. _[earned 2026-08-12]_

**The next case arrived and the tell did not fire, which is the finding.** A rename's inverse is a rename — the same shape of thing, nothing created and nothing destroyed — so the earlier rule says nothing about it, and the answer is still *not on the stack*. Two reasons, and the first is a defect to prevent rather than a feature to decline:

- **A rename is a file operation plus a change to every document that referenced it, and the second half lands on the undo stack by default.** Write the fixup through `edit` and Ctrl-Z puts every path back to the old name and leaves the file at the new one — the exact broken state the feature existed to remove, produced by the most natural implementation of it. So the fixup writes to disk **outside** the transaction API and tells the store afterwards through `adoptFromDisk`, which is what actually happened: the file changed underneath the editor. **Whenever a gesture is a file operation with document changes attached, check what the undo stack quietly picked up.**
- **Undoing it properly would put the first *fallible* step on that stack.** Reversing both halves means renaming back, and the old name may be taken again or the file may be gone. Every entry on this stack today is a patch application that cannot fail; one press that usually reverses a change and occasionally reports an error is a different key wearing the same shortcut.

So the rule generalises past its original wording: **an operation belongs on this stack when its inverse is patches over documents this window is holding — not when its inverse merely looks symmetrical.** _[earned 2026-08-12]_

**Third payoff, and the one that made the bet look cheap: a continuous gesture.** Dragging a sprite around the viewport writes a position on every pointer move, and it is *one* press of Ctrl-Z — because a drag is a merge key held open and sealed on release, which is the machinery a text field already used for a run of keystrokes. No undo code was written for it. Two things a gesture adds to the rules above. The merge key has to carry the entity's id as well as the field, or dragging one sprite and then another is one step. And the recipe re-finding its target by id stops being a precaution and becomes load-bearing: a drag lasts seconds, which is long enough for a text editor to save over the file underneath it. _[earned 2026-08-12]_

_[earned 2026-08-11]_

_[earned 2026-08-11]_

### D18: Autosave, debounced, with no dirty state and no save button

Every change is written to its file after a short quiet period; there is no save action and nothing models a document as modified-but-unsaved. **Reason:** the folder is the database (D3) and there is no open-document concept to hang a save on — a save button would imply a dirty state that nothing else in the kernel represents, and the human would have to learn a rule the architecture does not actually have. The debounce is 200ms and is spent out of the human's one-second budget (`editor-ui` U6). Two consequences worth knowing before relying on it: a write that is refused is **not** retried on a timer (a service that refused once will refuse again, and a silent retry loop is an editor hammering a service nobody has told anything is wrong) — it is recorded against that file and tried again on the next edit; and while a file has a change the folder has never accepted, re-reads of it are ignored, because taking one would discard the human's work *and* the only sign on screen that anything went wrong. _[earned 2026-08-11]_

### D19: When the file and the editor disagree, disk wins — and it is said out loud

A `.meta` changed in a text editor replaces what the editor is holding, rather than the editor refusing to write and reporting a conflict. **Reason:** the alternative is the editor arguing with Notepad, and it loses — the folder is the database, and a tool that refuses to proceed because a file changed underneath it is a tool that has to be argued with before it will do its job. The two residuals are accepted knowingly rather than discovered: a text-editor save landing inside the editor's ~200ms write window loses, and an undo of a field that a text editor has since changed puts that field back to what it was, because patches are per-field and undo means "put this back". _[earned 2026-08-11]_

### D8: A Node sidecar owns the filesystem; the editor is a local web app

Chokidar watcher + REST for JSON read/write + WebSocket for change events + static asset serving. **Reason:** browsers cannot watch folders. The File System Access API is Chromium-only, permission-prompty, and cannot push change events — so the "save a PNG and watch it appear" workflow is impossible in-browser and trivial with a small Node process. _[seeded 2026-08-11, report §8]_

**Confirmed in build.** The watcher and a read-only tree endpoint were built first, with chokidar 5 on Node 24 and Node's built-in `node:http`. No web framework was added: one read-only route is ~20 lines by hand, and Fastify is worth proposing only when the write API and static asset serving arrive. The WebSocket is likewise deferred until an editor exists to receive events — until then the terminal is the subscriber. Sequencing note for a fresh session: watcher + one GET endpoint is a complete, verifiable first feature; do not build the whole D8 surface at once. _[earned 2026-08-11]_

**Amended: server-sent events, not a WebSocket.** The change feed is `GET /events`, an SSE stream. **Reason:** changes only ever travel one way — the sidecar tells the editor what moved — and SSE is the shape of that problem exactly. It costs no dependency (a WebSocket server needs `ws`), and the browser's own `EventSource` reconnects by itself, which is reconnect logic not written and therefore not wrong. The editor→sidecar direction is REST, where it was always going to be. Revisit only if something genuinely needs the editor to push a stream back; nothing does yet. Two things that had to be got right: the server must end open streams before `close()`, or shutdown waits forever on a response that by design never ends; and the heartbeat timer must be `unref`'d, or an idle stream keeps the process alive. _[earned 2026-08-11]_

### D9: One command starts everything

`npm run editor` starts Vite and the sidecar concurrently. **Reason:** the editor must be ordinary software that runs without AI from day one. If starting it requires a session, the human cannot author content on their own schedule, and the whole division of labor stops working. _[seeded 2026-08-11, report §8]_

**Partially discharged.** `npm run sidecar` exists and takes the project folder as its one argument; `npm run editor` lands when there is a Vite app to start. Two things the one-command rule implies in practice, learned by writing it: the command must **print the absolute path it resolved**, because "which folder is it actually watching" is the first question a human asks and a relative argument does not answer it; and every refusal to start must be **one plain sentence naming the path or value at fault**, never a stack trace. Config resolution is therefore a pure function returning a result object, with the process exiting in the entry point only — that is what makes the refusals testable. _[earned 2026-08-11]_

**Discharged.** `npm run editor -- <project-folder>` starts both halves. Three things settled by building it:

1. **One process, not two.** The launcher starts the Vite dev server through its JavaScript API and the sidecar through an exported `startSidecar`, both inside itself. A supervisor spawning two children would have to kill a process tree on Ctrl-C, which is where this goes wrong on Windows — an orphaned server holding the port, with the next start failing for a reason that looks nothing like the cause. Keeping both in one process also removed the need for `concurrently` and `cross-env`.
2. **Validate before starting anything.** Config resolution runs first, so a missing folder or a taken port refuses while nothing is running. Vite comes up before the sidecar only so its address can appear in the sidecar's banner — one banner naming the folder, the editor URL, and the tree URL, rather than two interleaved ones.
3. **The project folder falls back to `KERNEL_PROJECT`**, command line winning, mirroring how the port already worked. This is what lets a test harness or a shell profile point the editor at a folder without putting it on the command line — the browser suite depends on it.

_[earned 2026-08-11]_

### D10: Sidecar first, Tauri later

Ship the web-app-plus-sidecar shape; wrap in Tauri only when a genre editor matures and wants to be double-clickable, swapping the sidecar's HTTP calls for Tauri's fs/watch APIs behind the same interface. **Reason:** the sidecar keeps the dev loop inspectable — Playwright drives it, `curl` pokes it, every boundary is observable. Deferring the wrapper costs nothing because the interface is the same; adopting it early costs verification. _[seeded 2026-08-11, report §8]_

### D11: The kernel persists; the genre layer regenerates

The kernel is the genre-agnostic ~60%: scaffold, sidecar, asset browser, meta system, serialization core, undo/redo, inspector framework, play-in-editor harness, test setup. Genre tooling is built fresh per game. **Reason:** undo and serialization are precisely the systems that must never be rewritten ad hoc, because that is where subtle bugs live. Genre tools are the opposite — a bespoke wave designer beats a generic timeline, and each is small enough to produce quickly against established kernel patterns. _[seeded 2026-08-11, report §5]_

### D12: The skills are the engine; the kernel repo is their cached output; the tests are the contract

Dual-write is definition-of-done, and the parity drill (regenerate from skills alone, run the real suite against it) is the mechanical proof the skills have not silently fallen behind. **Reason:** the two pure strategies both fail — maintaining the kernel forever ossifies it, and regenerating from nothing re-pays the hard-systems tax every game. Holding both requires that the knowledge live somewhere the code cannot drift away from unnoticed. Costs ~10–15% overhead per session. _[seeded 2026-08-11, report §5]_

**The protocol, corrected by running one. "From skills alone" is the wrong bar, and holding it hides the finding that matters.**

The drill's inputs are **the skills *and* the test suite**, because that is what a customer receives — the report's own distribution section says the suite ships with the skills as the warranty, and clean-room means no access to the reference *kernel*, not no access to the tests. Running it the stricter way, with the tests withheld, is still worth doing once: it separates two failure modes that look identical in a combined run.

- **What the skills carry on their own: the architecture and the behaviour.** A runtime regenerated with no sight of the code or the tests obeyed the layer boundary, put the registry beside the entity that validates against it, kept resolution in the format, made the loader fatal only about the document it was asked for, and got the frame arithmetic's `n-1` gaps right.
- **What they do not carry, and should not: identifiers.** Nearly every failure in that run was a function called something else. Writing names into the skills would be a second copy of the interface that drifts from the tests the first time either moves — and the tests are the contract (this decision's own title). **A name's home is the suite.** The exception is a value that ends up *on disk in somebody's project folder*, which is data rather than interface — see `text-formats` T17.

**And scope a drill to a layer only where that layer's consumers are tests.** Regenerating `runtime/` judged the ten modules whose contract is Vitest; it could say nothing about the renderers, whose only contract is the browser suite and whose caller is an `editor/` that was not being regenerated. A layer-scoped drill silently narrows to the part with unit tests, so decide that scope deliberately and say what it left out — the alternative is a green-looking drill that never touched the half most likely to have drifted. _[earned 2026-08-12, first drill]_

### D14: AI writes code that writes files; only humans author data through the tools

The narrow exception is migration scripts — a one-off format converter is tooling, not authorship. **Reason:** the moment level data is authored by prompt rather than by tool, the data stops reflecting human intent and the anti-slop guarantee is gone. This is the thesis of the whole methodology, not a preference. _[seeded 2026-08-11, report §3/§9]_

### D15: One editor UI stack across both dimensions

React + docking layout + Zustand/immer + Zod, with only the viewport differing between 2D and 3D kernels. **Reason:** the panels, trees, and inspectors are identical work in both; sharing the stack means the 3D kernel inherits a proven shell and the hard 3D-specific problems (camera, raycast selection, gizmos) get the full budget. Detailed idioms belong to `editor-ui`; the constitution only fixes the stack. _[seeded 2026-08-11, report §6/§7]_

**Confirmed in build.** The shell is React 19 + Vite 8 + `dockview-react` 8 (the React binding package in v8; `dockview` and `dockview-core` arrive underneath it). Zustand and immer were deliberately **not** installed with the shell: there is no document state for them to hold until the first panel that edits something, and adding the undo machinery before there is anything to undo would fix its shape against an imagined document. They land with the first editing panel, not before. _[earned 2026-08-11]_

---

## Gotchas

### G1: Serialization drift is the top structural risk, and it must be fenced before the second tool exists

Save and load quietly disagree; nothing errors; corruption surfaces later as lost work. **Fix/policy:** the Zod-single-source-of-truth rule (D6) plus a standing `load(save(x)) === x` round-trip test for every schema, running on every change. The ordering matters — the tripwire must exist before the second tool is written, because the drift becomes expensive exactly when there are multiple writers of the same format. _[seeded 2026-08-11, report §4/§13]_

### G2: A tool that mutates the document outside the transaction API appears to work

Nothing fails at write time. The damage shows up as undo skipping a step or restoring stale state, far from the tool that caused it. **Fix/policy:** treat "does this mutation go through the transaction API?" as a review question for every session that touches the document, and prefer making the direct-mutation path structurally unavailable over documenting that it shouldn't be used. _[seeded 2026-08-11, report §5/§13]_

### G3: Quality erodes invisibly when nobody reads the code

Duplication, dead code, and creeping complexity are not caught by behavior tests — the editor keeps working while future sessions get slower. **Fix/policy:** a scheduled gardening session every few features (audit the codebase against the skills' conventions, second-model review, refactors behind the same green gate). The smoke alarm is quantitative: if session cost or time for similar-sized features trends upward, the garden needs tending. _[seeded 2026-08-11, report §13]_

**Run for the first time after four features, and the shape that made it safe is worth keeping.**

**The gate is "no test assertion changes."** Every refactor lands behind the full suite, and afterwards `git diff --stat -- tests/` is read as its own check: anything but a mechanical rename means behaviour moved, and the session stops rather than adjusting the test. That single rule is what separates gardening from rewriting, and it is checkable rather than aspirational.

**What a survey actually turns up, in rough order of value.** Worth using as the checklist, because none of it is visible from the outside:

1. **A constraint recorded without a measurement.** The most valuable find by a distance: a rule that had shaped two design decisions turned out to be false when tested (`text-formats` T14). Anything in these skills phrased as "X is impossible" and not marked as measured is a gardening candidate — cheap to check, and the cost of believing it compounds.
2. **Small presentational components duplicated per panel.** Four copies of `Section`/`Field`/`Note`/`Row` had accumulated, one per inspector body, because each was added by the session that needed it and no test can see the duplication. Check on sight whenever a third panel of a kind exists.
3. **Two hooks that are the same mechanism with different vocabulary.** Following a texture reference and following a prefab reference were ninety identical lines. The tell is a second file whose *comments* echo the first one's.
4. **Numeric claims in prose.** "Every one of the four actions", "the only code that…", "a one-line branch" — all true when written and all silently false a feature later. Grep for them; in a codebase whose owner never reads it, a stale comment is worse than none.
5. **A file whose name stopped describing it**, and test hooks named after half of what a control does.

**What to leave alone.** Long files that are well-sectioned. Splitting `ViewportPanel.tsx` for line count would scatter something that currently reads top to bottom, and length is not the smell — a file doing several unrelated things is. _[earned 2026-08-12]_

**Run a second time, four features later, and the useful result was the measurement rather than the refactors.**

**The smoke alarm was measured, and it is quiet.** Lines added per minute of wall clock between feature commits, which is crude and includes the human's review time, but is the number this gotcha has always been phrased around: drag-to-place 35, make-a-level 39, prefabs 38, play mode 37, export 38. **Five consecutive features inside a 10% spread with no drift.** The early sessions ran at 71–102 and are not comparable — greenfield scaffolding with nothing to integrate against is a different activity, and treating it as the baseline would make every later session look like decay.

**So the cadence moves to roughly every eight features, and the trigger becomes the number rather than the count.** Garden when the rate falls meaningfully below the high-30s, or after eight features, whichever comes first. Gardening on a fixed short interval spends real budget hunting duplication that has not accumulated yet — this pass found one comment worth fixing — and, worse, it invites changing things to justify the session. **Measure before surveying, and be willing to conclude that the garden did not need tending.**

**Two corrections to the checklist above, both from following it.**

- **Item 3's tell fires on correct duplication as often as on drift, so the survey has to finish the reasoning.** Two files doing near-identical arithmetic with echoing comments (`editor/shell/play-comparison.ts`, `kernel-2d/tests/instruments/drawn-comparison.ts`) turned out to be unmergeable: they compile in *different TypeScript projects*, the only module both can reach is `runtime/`, and a comparison against an editor inside the shipping layer is precisely what D1 forbids. Twenty-five duplicated lines is the right price. **Record a correct duplication where the next pass will find it**, or every future survey re-opens it and re-derives the same answer.
- **A count carried forward in a checklist ages exactly like a count carried in a comment.** The previous pass left "four descriptions of the same three failures, and the export added a fifth". Checked: there are **two** (`scene-assets.tsx`, `scene-prefabs.tsx`). The export command consumes the runtime's one union and applies its own policy, which is what D13 designed; the project inspector answers a different question in a deliberately flatter shape. The item was real and half its scope was imagined — so **a survey verifies its own inherited list before acting on it**, which costs one read per item and is the difference between gardening and rearranging.

**The sharper form of that item, for whoever takes it next.** The duplication that actually matters there is not the two type unions, it is that **three of the sentences a human reads are byte-identical across `runtime/scene/load-scene.ts` and the two editor files.** Improve the wording in one and the editor and the exported game start describing one failure two ways. The constraint that made the split correct — the runtime may not import the editor — does not forbid the reverse, and `describeLoadProblem` is already exported and already consumed by the export command. Left undone deliberately this pass; it is a change with a visible surface, and the gate here is that nothing observable moves. _[earned 2026-08-12]_

### G4: "Just generate this one level" breaks the guarantee retroactively

The pressure appears when a tool doesn't exist yet and the content is needed now. **Fix/policy:** the rule is absolute (D14) — the answer to missing tooling is the tool, not the content. Note the failure is retroactive: once prompt-authored data is in the project, no later tool work restores the provenance of what shipped. _[seeded 2026-08-11, report §3]_

**The generated-sample case, and where its line sits.** Sample content for testing a tool is legitimate and gets generated by a **script**, not by a prompt — deterministic, re-runnable, and reviewable as code. Three rules learned writing one. It must **refuse to touch any file lacking a generated marker**, which is what makes it safe to re-run against a folder somebody has since worked in. It must produce **byte-identical output on every run**, so seeded randomness and a passed-in date, never `Math.random()` or `new Date()` at the point of use — otherwise re-running churns the folder and the noise hides real changes. And it must not **fake a format that has no schema yet**: placeholder JSON says it is a placeholder, because plausible-looking contents are how a wrong format quietly becomes precedent. The same restraint applies to file types — no fabricated `.psd` or `.aseprite`; a file claiming to be something it is not is a trap for whoever opens it. _[earned 2026-08-11]_

### G5: Scope creep toward "building an engine" has no natural stopping point

**Fix/policy:** the genre spec is the fence — if a tool isn't justified by a noun in the spec, it isn't built this cycle. The kernel will become an engine over time; the requirement is that it happen by accretion from shipped needs rather than by anticipation. _[seeded 2026-08-11, report §13]_

### G6: A file watcher's default write-settling delay is far past "noticed within a second"

Chokidar's `awaitWriteFinish` is what stops a half-written PNG being announced while Photoshop is still saving it — but its default `stabilityThreshold` is 2000ms, which silently turns a sub-second requirement into a two-second one. **Fix/policy:** set it explicitly (200ms was measured to land the notice ~250ms after the save completes, and is still far longer than the gap between writes in a slow save). Never leave it at the default, and never turn it off — off means the editor sees truncated files. _[earned 2026-08-11, chokidar 5.0.0]_

### G7: No operating system reports a rename as a rename

A rename arrives as an unlink of the old name plus an add of the new one. Any code that tries to present renames as a single event is inventing a correlation the OS did not give it. **Fix/policy:** report the two events honestly and say so in the UI/banner text, so the human is not confused by seeing their one action produce two lines. Chokidar's `atomic` option coalesces unlink+add **at the same path** (editors that save via temp-file swap) — it does not and cannot pair up a real rename. _[earned 2026-08-11, chokidar 5.0.0]_

**Still true of the watcher, and it stopped being the last word — because the tool can rename the file itself.** The escape is not a better correlation, it is *being the one who did it*: an editor that performs the rename knows both paths before either event exists, so it can move the sidecar with the file, repair every reference, and never consult the watcher at all. The two events still arrive and are still uncorrelatable; nothing depends on them any more.

Two consequences worth having in advance. **The service must then ignore its own file operations** — it is watching the folder it is writing into, and its create-when-missing rule will happily fill the gap between the two renames with a fresh id (D22's ordering note). And **the gotcha's scope has to be re-read rather than deleted**: it still governs everything done outside the tool, which is where the loss now lives. Whenever a limitation is dodged by moving *who performs the action*, the limitation is intact and the entry that describes it needs a scope line, not a strike-through. _[earned 2026-08-12]_

### G9: A service that writes into the folder it watches will see its own writes, and that is a loop unless something breaks it

Creating a `.meta` in response to a file appearing produces a change event for the `.meta`, which is another file appearing. **Fix/policy:** the loop is broken by the *rule*, not by a flag — a `.meta` is not an asset, so the second pass through does nothing, and the sidecar never gives a `.meta` a `.meta`. Two things make this reliable rather than lucky. Write the termination argument into the code as a comment where the handler is, because a later session widening "what counts as an asset" needs to see what it would break. And **run the bulk sweep before the watcher starts**, not after: a few hundred sidecars created with the watcher already listening is a few hundred change events describing the service's own work, which the editor then treats as the human having done something. A test asserts the folder listing is unchanged after the dust settles, which is the only form of "it does not loop" that is actually checkable. _[earned 2026-08-11]_

### G10: The editor writing a file feeds itself, and it converges for a reason worth writing down — but only after a race is closed

The editor writes a `.meta`, the watcher reports the change, the folder is re-read, the settings are re-fetched, and the answer arrives back at the editor. Same shape as G9, one layer up, and it terminates for a completely different reason.

**Why it converges: the round trip is identity.** What comes back is exactly the document that went out, so taking it changes nothing, so nothing schedules a second write. The cycle has a fixed point and the very first iteration is already sitting on it. What guarantees the identity is the round-trip property the format is already tested for — `load(save(x))` equals `x` (G1, `editor-verification` V7). **That test is therefore doing double duty**, and a session that breaks it does not only break the format: it turns this into a loop that rewrites the same file for ever. Say so in a comment where the handler is, because nothing else connects the two.

**The race that converging does not close.** A read *issued* before the write can be *answered* after it, and it then carries the file's contents from before the write. Nothing about that answer looks stale — it is an ordinary reply to an ordinary question — so taking it reverts the change and then writes the reversion back to disk. On screen the setting simply does not stick, and nothing anywhere reports an error. **Fix/policy:** the guard has to be on when the read *started*, not on when it arrived; by arrival the write is long finished and a stale answer is indistinguishable from a fresh one. In practice: hand out a token before asking, hand it back with the answer, and discard any answer whose token predates the last write to that path. Discarding costs nothing, because the write's own watcher event triggers a read that is unambiguously newer.

Two more guards belong beside it, for the same reason and with different timing: ignore a re-read while a write for that file is queued or in flight (it is the file as it was before the keystroke), and ignore one while a write for that file has been refused (the editor is holding a change the folder never accepted). _[earned 2026-08-11]_

### G11: A file re-saved with the same number of bytes is invisible to anything watching its size

Re-exporting a PNG from Photoshop very often lands on the identical byte count. Anything that caches a file — a preview keyed by URL, most obviously — will keep showing the old contents, and the human's edit appears simply not to have happened. **Fix/policy:** carry the modification time alongside the size wherever the editor describes a file (`FileNode.mtimeMs` in the file-tree format), and use it as the cache key. Do not round it to whole milliseconds: some filesystems report finer than that, and rounding is the schema quietly disagreeing with `stat`. A change-event counter is the tempting alternative and is worse — it is a second piece of bookkeeping that goes wrong when an event is missed, whereas the timestamp is simply true. _[earned 2026-08-11]_

### G12: `structuredClone` throws on a draft, so a recipe that copies something must copy it another way

Duplicating an entity inside a transaction is the natural place to reach for `structuredClone`, and it fails — the recipe's argument is an immer draft, which is a Proxy, and a Proxy cannot be structurally cloned. It compiles, it type-checks, and it throws the first time the button is pressed with a `DataCloneError` that names nothing you wrote.

**Fix/policy:** copy through JSON, and put the copy function next to the *format* rather than in the panel with the button. Both halves matter. JSON is faithful here by construction — a document is JSON by definition — and it reads through a draft the way any property access does. And what has to survive a copy is a fact about the format, not about the tool: everything, including component types this kernel has no schema for. A copy that quietly dropped one would look exactly like working, and the loss surfaces weeks later to somebody with no reason to suspect the Duplicate button. _[earned 2026-08-12, immer 11]_

### G13: A browser refuses to load a module script from `file://`, so a page's own "you have to serve me" message cannot be in the module

A web export cannot be opened by double-clicking, and the reasons stack up in an order that matters. Every `file://` document gets its own opaque origin, so `fetch` of the level beside the page is refused; a texture loaded from a sibling file would be refused again on its way to the graphics card as a cross-origin image; and **before either of those, a `<script type="module">` is not loaded at all.** So the honest sentence explaining the situation — the one thing standing between the human and a black rectangle with an error in a console nobody has open — must not live in the module. It never runs.

**Fix/policy:** put the check in a plain inline `<script>` in the page, which is the one thing that still executes from `file://`, and leave a note in the module saying why it is not there. Bundlers leave a non-module inline script alone, so it survives the build. The general shape is worth recognising: **a diagnostic for "the environment cannot run me" has to be written in something the broken environment can still run.** Check it by actually navigating to the `file://` URL in the browser suite; reasoning about it gets the ordering wrong.

The corollary for the export as a whole: the folder is served or uploaded, never double-clicked, and that has to be said in the command's output and in the human's page rather than discovered. _[earned 2026-08-12, Chromium]_

### G8: Path separators leak the authoring machine into the data

Windows produces `assets\textures\knight.png`, macOS produces `assets/textures/knight.png`, and the same project then serializes two different ways depending on who saved last. **Fix/policy:** one conversion helper at the boundary, forward slashes in every path that reaches JSON, a terminal, or a test, and an explicit assertion in the test suite that no emitted path contains a backslash. This has to be settled in the first format, because every later format inherits the convention. _[earned 2026-08-11]_

---

## Contracts

Contracts are referenced as file paths, never paraphrased as prose. Read the file; don't trust a summary of it.

**Exists now:**

- `kernel-2d/CLAUDE.md` — project law: division of labor, AI-generated-content marking rules, folder map, definition of done, session conduct. Outranks this skill on process; this skill outranks it on architecture rationale.
- `kernel-2d/docs/ai-game-tooling-report.md` — the source these decisions were seeded from. §4 architecture, §5 kernel/genre split and Option C, §8 sidecar shape, §9 the vibe-coding contract.
- `gamedev-skills/vendor/phaser4/` — vendored Phaser 4 ground truth. Owned by `phaser4-runtime`; referenced here only so the kernel's runtime layer knows where authority lives.
- `kernel-2d/sidecar/config.ts` — how the sidecar is pointed at a project folder, and every reason it will refuse to start. Loopback host and default port live here.
- `kernel-2d/sidecar/watcher.ts` — the change-event shape (`FileEvent`) and the write-settling policy. This is the payload the WebSocket will carry when it lands, so change it deliberately.
- `kernel-2d/sidecar/tree-schema.ts` — the file-tree format served by `GET /tree`. Owned by `text-formats`.
- `kernel-2d/sidecar/status-schema.ts` — the status format served by `GET /`: which project is open and what else this sidecar serves. The first format read by both halves of the system.
- `kernel-2d/sidecar/event-schema.ts` — the change format carried by `GET /events`, and the home of the change vocabulary (D16).
- `kernel-2d/sidecar/feed.ts` — the hand-off from the watcher to everything listening: the terminal, and every open editor window.
- `kernel-2d/runtime/formats/meta-schema.ts` — `AssetMetaSchema`: the `.meta` sidecar format, the extension→type vocabulary, and the defaults factory every writer builds from. Lives in `runtime/` because the shipping layer reads it (D20). Owned by `text-formats`.
- `kernel-2d/runtime/index.ts` — the runtime's public surface: what the editor may import, and the one-way arrow of D1/D2.
- `kernel-2d/runtime/textures/frames.ts` — what a slice means, in pixels. One definition, shared by the renderer that cuts the frames and the overlay that draws them (D2).
- `kernel-2d/runtime/textures/import-settings.ts` — what a setting *does*: filtering and slicing applied to a live texture, and the filter read back off it rather than echoed.
- `kernel-2d/runtime/preview/texture-view.ts` — one renderer for the life of the window, and its report of what it actually drew.
- `kernel-2d/sidecar/asset-files.ts` — D21 in four lines: the whole of what this service will read out of a human's project folder.
- `kernel-2d/tests/architecture/boundaries.test.ts` — D1 as a test rather than as a promise.
- `kernel-2d/sidecar/meta-view-schema.ts` — what `GET /meta` answers with, including the two answers that are not a meta: there isn't one, and there is one this editor cannot read.
- `kernel-2d/sidecar/meta-files.ts` — the whole of D17 in one file: create-when-missing, the startup sweep, the one write the editor can ask for, and the path validation everything from the browser passes through.
- `kernel-2d/sidecar/server.ts` — the HTTP surface as it currently stands: `GET /`, `GET /tree`, `GET /events`, `GET /meta?path=…`, `GET /document?path=…`, `GET /asset?path=…`, the two PUTs to `/meta` and `/document`, `POST /document`, `POST /move` and `POST /delete` — the only non-GETs this service answers at all, one route per intent so none of them can do another's job.
- `kernel-2d/sidecar/document-view-schema.ts` — the registry of document formats the editor reads and writes, keyed by the `format` the document itself carries (D22), and the answer `GET /document` gives about one.
- `kernel-2d/sidecar/document-files.ts` — two of D22's six lines: the create and the replace kept apart, and every guard that keeps each of them from doing the other's job.
- `kernel-2d/sidecar/file-operations.ts` — the other two: the move and the delete, the ordering of a file and its `.meta` in each, the rule that stops the service acting on its own writes, and the one check-then-act in the service, admitted rather than hidden.
- `kernel-2d/sidecar/file-change-schema.ts` — what a move or a delete answers with, and why refusal has no spelling in it.
- `kernel-2d/editor/shell/useFileMoves.ts` — the fixup as a gesture: flush, plan, refuse before anything moves, rename, rewrite, report. The comment on why none of it goes through `edit` is D7's newest line.
- `kernel-2d/editor/shell/references.ts` — D5's fixup: which documents point at a file, and the same documents with its new path in. `path` moves, `id` never does.
- `kernel-2d/runtime/formats/scene-schema.ts` — `SceneSchema`: the flat entity list, the transform, the open component map, and the registry of components the kernel knows. Owned by `text-formats`.
- `kernel-2d/runtime/formats/prefab-schema.ts` — `PrefabSchema` and the resolution of D25. Imports the scene's registry and is never imported back, which is the whole of why these are two files (T14).
- `kernel-2d/editor/shell/scene-prefabs.tsx` — D25 as built: which prefabs a level points at, read into the store, merged by the format's own function, and the three things that can be wrong with a reference.
- `kernel-2d/editor/shell/usePlacePrefab.ts` — one gesture, two places to reach it from, and why the prefab comes from the store rather than from what the level already references.
- `kernel-2d/runtime/scene/scene-view.ts` — the second renderer, and its report of which entities it drew, where, through which camera, and how much level there turned out to be.
- `kernel-2d/runtime/scene/load-scene.ts` — **D2's second sentence**: a level opened by the runtime itself. The `ProjectReader` seam (D26), what is fatal and what is merely named, and the sentence each problem gets. Also the worked example of D16's Node-compilable constraint.
- `kernel-2d/runtime/scene/scene-request.ts` — why two interfaces live apart from the renderer that consumes them: the compiler boundary of D16, stated where somebody would otherwise move them back.
- `kernel-2d/editor/shell/project-reader.ts` — the loader's one adapter to the development service, and the note about which half of it the browser never exercises.
- `kernel-2d/runtime/web/start-game.ts` — the far side of D26: a game that reads its own files with `fetch`, boots one renderer, frames its level and says what went wrong in the runtime's own words. The whole of what an exported folder runs.
- `kernel-2d/runtime/web/index.html` — the only HTML the runtime owns, and the home of G13's inline sentence.
- `kernel-2d/runtime/formats/project-schema.ts` — `ProjectSchema`: the startup scene, why it is a path with no id beside it, and what is deliberately not in this file yet. Owned by `text-formats`.
- `kernel-2d/runtime/scene/scale-steps.ts` — the zoom ladder, in the layer that ships (D20's third case).
- `kernel-2d/runtime/scene/drawn-in-scene.ts` — the renderer's report, in the level's own units. One projection, called by the editor's viewport and by the exported game.
- `kernel-2d/scripts/export.ts` and `kernel-2d/scripts/export/` — **D13's web half**: `config.ts` (where it may write), `plan.ts` (what goes in and every refusal), `write.ts` (the four steps and their order), `manifest-schema.ts` (the record that makes a second export safe), `editor-markers.ts` (D1 checked on the artefact), `project-reader.ts` (the third reader).
- `kernel-2d/scripts/serve-folder.ts` and `kernel-2d/scripts/serve.ts` — how an export is looked at, and the read privilege of a folder served locally. Deliberately not part of an export.
- `kernel-2d/editor/shell/play-mode.tsx` — what Play does before it reads the file, and why a refused save stops it.
- `kernel-2d/editor/shell/play-comparison.ts` — "what I see matches what the editor was showing me", as arithmetic over two reports from one renderer.
- `kernel-2d/runtime/scene/entity-layer.ts` — documents into drawn objects: matched by id, updated in place, depth from list order.
- `kernel-2d/runtime/scene/coordinates.ts` — scene space (y-up, bottom-left), the camera (D23), and how the two become screen space. One definition, no Phaser.
- `kernel-2d/editor/shell/scene-view-context.tsx` — where the view lives: one camera per scene for the life of the window, and the three conditions a scene has to satisfy before it is framed.
- `kernel-2d/editor/shell/useSceneGestures.ts` — middle-drag, space-drag, wheel-to-zoom, and the two framing keys.
- `kernel-2d/editor/shell/scene-assets.tsx` — D5 as built: resolve by path, compare the id, report rather than veto.
- `kernel-2d/editor/store/document-disk.ts` — the editor's write for documents, beside `meta-disk.ts`.
- `kernel-2d/editor/store/documents.ts` — **the transaction API** (D7): the store, the two doors, the single time-ordered undo stack, the coalescing rule, the debounced save, and the convergence argument of G10 written where the handler is. Nothing else in the editor may change a document.
- `kernel-2d/editor/store/open-documents.ts` — the one store per window, and the hooks panels read it through.
- `kernel-2d/editor/store/meta-disk.ts` — the editor's whole write privilege, in one function.
- `kernel-2d/sidecar/start.ts` — bringing the sidecar up as a library rather than as a command, which is what lets the editor launcher host it in-process (D9).
- `kernel-2d/sidecar/ignore.ts` — what the sidecar never lists and never watches.
- `kernel-2d/scripts/editor.ts` — the one command (D9).
- `kernel-2d/editor/` — the editor shell. Owned by `editor-ui`; referenced here only so the boundary is visible from the constitution.

**Not yet written** — these are the kernel's core contracts and land as the corresponding sessions build them. Until a path appears here, the contract does not exist and must not be assumed:

- `PrefabSchema` — reusable entity templates. The document endpoint is already shaped to take it: adding one is a schema plus a line in the registry.
- Deleting a *folder* from the editor. Renaming and moving take folders; deleting is one file at a time, because "delete this folder and everything under it" is a blast radius and the question the editor asks first — what still uses this? — stops having a short answer (D22).
- Parenting. The entity list is flat and ordered, which is the shape a `parent` field can be added to without a migration; nothing can set one yet, so nothing carries one.
- Repairing a reference to a file that moved **outside** the editor. The fixup exists for a move the editor performed, where both paths are known before either watcher event arrives (D5, G7). Outside it there is still nothing to correlate a disappearance with an appearance, and the settings are gone at the next start.
- A desktop export. D13's other half. The web export exists and is a static folder, so a native shell is a wrapper over it rather than a second build of the game.
- Anything that moves. There is no update loop, no input handling and no component but `sprite`; play mode and an exported game both load and draw, and stop there.
- More than one level in a game. `project.json` names a startup scene and nothing can reach a second one, so the export's transitive closure is complete by accident rather than by design. The day a level can name another, the closure walk in `plan.ts` is the thing that follows it.
- Anything about making an export small: no minification (deliberately off, so D1's marker check stays trustworthy), no packing, no atlases. `game.js` is a few megabytes.
- Creating `project.json` from inside the editor. It is edited when it is there, the sample generator writes one, and the export refuses by name when it is not — but nothing in the editor will make one.
- One vocabulary for the same three failures. The editor's edit-mode resolution and the runtime's loader each describe "missing", "unreadable" and "not the file this was written against" in their own words, in `scene-assets.tsx`, `scene-prefabs.tsx` and `load-scene.ts`. **Surveyed 2026-08-12 and left standing deliberately.** The union *shapes* genuinely differ and should — the editor holds a folder listing and can say "that file is not in the project", which the runtime has no way to know. What is not defensible is that three of the **sentences** are byte-identical across the boundary; the fix is the editor's two describe functions delegating to the runtime's `describeLoadProblem`, which is already exported and already consumed by the export command. It has a visible surface, so it belongs to a session whose gate allows one.
- Inspector controls generated from a Zod schema. The texture settings and the entity fields are both hand-written; there are now two inspectors to generalise *from*, which is the point at which it becomes worth doing.
