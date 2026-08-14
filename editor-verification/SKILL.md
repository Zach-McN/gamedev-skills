---
name: editor-verification
description: Testing recipes and invariants for game editors — how the editor's correctness is verified through tests, property checks, and the invariants that must always hold. Consult this skill whenever writing tests for editor code, defining an invariant the editor must preserve, setting up verification for the kernel, or deciding how a piece of editor behavior should be proven correct.
---

# editor-verification

The verification handbook for the editor — the testing recipes, harnesses, and invariants that prove the kernel behaves correctly. This skill defines how editor correctness is established and re-established: what gets tested, which invariants must always hold, and how the test suite is structured so a fresh session can trust it. Alongside the skills themselves, the test suite is what lets the kernel regenerate. Content is earned from real verification sessions, never invented.

Targets Vitest 4.x on Node 22+.

## Decisions

### V1: The human's acceptance criterion is transcribed into a test, in their units

"I drop a PNG in and the terminal notes it within a second" becomes an assertion that the elapsed time between the write and the event is under 1000ms. **Reason:** acceptance criteria stated as observable behavior are already tests; restating them as implementation checks loses the only requirement that was actually agreed. It also makes the timing budget visible, so the next session cannot quietly spend it. _[earned 2026-08-11]_

### V2: Filesystem behavior is tested against a real filesystem and real OS events, never a mocked watcher

**Reason:** a mocked watcher tests the mock. Every property worth knowing here — whether a rename produces one event or two, whether a partial write is announced early, whether the ignore rules actually take effect — is a property of the operating system, and a fake cannot have it wrong in the same way the real thing is wrong. The cost is real: the watcher suite is the slow part, and that is the correct trade. _[earned 2026-08-11]_

### V3: Test project folders are built on disk at test time, not committed as fixture files

A fixture builder creates a temp folder from a map of `{ 'assets/textures/knight.png': 'contents' }`, and a key ending in `/` makes an empty folder. **Reason:** the ignore rules cover `node_modules`, `.git`, and `dist` — exactly the folders the repo's own `.gitignore` would strip out of a committed fixture, silently gutting the tests that check them. Building at test time also makes each test's inputs readable in the test itself instead of somewhere else in the tree. The builder lives in `tests/fixtures/`. _[earned 2026-08-11]_

### V4: The temp folder is `realpath`'d before use, and servers bind port 0

The system temp folder is commonly reached through a link, and watchers report the resolved path — comparing against the unresolved one produces failures that look like watcher bugs. Port 0 asks the OS for a free port, so a test run never collides with a sidecar the human has open. **Reason:** both are ways of making the suite independent of the machine it runs on, which is the precondition for the parity drill meaning anything. _[earned 2026-08-11]_

### V5: Waiting is done by polling for a condition, never by sleeping a fixed amount

A `waitFor(probe, description, timeout)` helper polls every 10ms and fails with the description on timeout. **Reason:** fixed sleeps are simultaneously flaky and slow — long enough to waste the suite's time, never long enough on a loaded machine. The named description is what turns a timeout into a diagnosis. _[earned 2026-08-11]_

### V6: Negative assertions need a positive event to anchor them

"The watcher says nothing about `node_modules`" is unprovable on its own — nothing has happened *yet* is indistinguishable from nothing will happen. **Fix:** write the ignored file, then write a watched file, wait for the watched file's event, and only then assert the ignored one produced none. The watched event proves the watcher had the opportunity. _[earned 2026-08-11]_

### V7: Every schema carries a round-trip test from the moment it exists

`load(save(x))` deep-equals `x`, compared against the original object. **Reason:** `editor-kernel` G1 — the tripwire has to be in place before the second writer of a format exists, because that is when drift starts and it is silent. See `text-formats` F1 for the way this test is commonly written wrong. _[earned 2026-08-11]_

### V8: The browser suite starts the editor with the same one command a human uses

Playwright's `webServer` runs `npm run editor`, pointed at a throwaway project through `KERNEL_PROJECT`, with `reuseExistingServer: false`. **Reason:** two payoffs for one line of config. The one command is then itself under test — if it breaks, the browser suite goes red rather than passing against a hand-assembled substitute that only the tests know how to start. And refusing to reuse an existing server stops a run from silently attaching to the editor the human has open, where every assertion about the project folder would be a lie. Both ports are shifted off their defaults for the same reason: a test run must not collide with a live editor. _[earned 2026-08-11]_

**Extended to every command the suite depends on, which is the whole point of the rule.** There are now two servers and a build step, and each is the command a human would run: the sample project is written by the real generator, the exported game is built by running `npm run export` **as a process**, and it is served by `npm run serve`. Three commands under test on every browser pass, for three lines of config.

Two practicalities. **The order is forced and is not obvious:** the project is written, the export is built from it, and only then can the static server be pointed at the folder — and Playwright starts its servers before anything else, so all of that has to finish while the config file is still loading. Running the export with `execFileSync` rather than by importing its functions is what makes that possible in a synchronous config, and it is the more honest test anyway. **Surface the command's own output on failure:** `execFileSync` puts stdout somewhere nobody looks, and a failure here stops the entire suite with "Command failed" explaining none of it. Catch it and re-throw with the captured output.

One thing to avoid: do not export a URL constant from `playwright.config.ts` for a spec to import. The spec would re-run the config's top-level work — writing the project, building the export — inside the test process. Put shared addresses in a module of their own. _[earned 2026-08-12]_

### V9: A negative UI state is produced by cutting the request, not by tearing down the service

The "sidecar is not answering" state is tested by aborting the `/api/` route in the browser and reloading — after first asserting the connected state on the same page. **Reason:** V6's anchoring rule, applied to the browser: the warning is only meaningful once the same test has seen the strip say "connected". Cutting the request at the browser is also far cheaper and more deterministic than stopping and restarting the real service mid-run. _[earned 2026-08-11]_

### V10: The browser suite builds its project with the real content generator

The throwaway project the browser tests open is the actual sample project, written by the actual generator, with the date pinned so re-runs produce identical bytes. **Reason:** two things get tested by one arrangement — the generator runs on every browser pass, and the panel is exercised against the folder the human will really be looking at rather than a three-file stand-in that never has a `.meta`, a nested folder, or a paired asset in it. Anything a test needs that the sample lacks is a sign the sample is too thin, not a reason for the test to build its own. _[earned 2026-08-11]_

### V11: Live-filesystem behaviour is tested by writing to the real folder mid-test

"I save a PNG and it appears" is asserted by writing the file straight into the watched folder from the test process, then asserting the row is on screen within the second. **Reason:** V2 carried into the browser — the whole chain under test is watcher, feed, stream, proxy and panel, and any stub replaces the part most likely to be wrong. The test cleans up by deleting the file, which doubles as the deletion assertion. _[earned 2026-08-11]_

### V12: "It never touched my file" is proved with the bytes *and* the timestamp

A test that asserts a file was left alone reads its contents and its `mtime` before and after, and asserts both are unchanged. **Reason:** contents alone pass for a file that was rewritten with identical bytes, which is not the same promise — a service that rewrites a file every startup churns the folder, breaks any external watcher, and would happily rewrite one whose contents it had *also* changed the day the writer gained a field. The timestamp is what makes "did not write" distinguishable from "wrote the same thing". _[earned 2026-08-11]_

### V13: A destructive behaviour is tested from both sides, in the same file

Deleting stranded sidecars at startup is asserted twice: that a start clears one, and that a running session does *not*. **Reason:** a rule of the form "only ever at X" is two claims, and the second one is the one a later session breaks by accident while making the first one more helpful. Written next to each other, the pair reads as the rule; written apart, the negative half looks like an oddity somebody can delete. Same shape as V6's anchoring, applied to a policy rather than an event. _[earned 2026-08-11]_

### V14: A browser test that changes the shared project puts every file back afterwards

Tests that edit settings snapshot every `.meta` in the sample folder before each test and restore anything that differs after it, deleting anything that appeared. **Reason:** V10's shared project is built once per run, and the suite runs single-worker in file order — so a test that leaves a changed file behind makes a *later* test's outcome depend on which files happen to run first. That is the worst kind of flake, because it gets diagnosed as a product bug in whichever test drew the short straw. Snapshotting the whole set rather than the files a test names is deliberate: a test that writes somewhere unexpected is exactly the one that would forget to declare it. _[earned 2026-08-11]_

### V15: A Vitest test of browser-side code lives outside the Playwright folder

`tests/store/`, not `tests/editor/`. **Reason:** Playwright's default match claims `*.test.ts` as well as `*.spec.ts`, so a Vitest file in the browser suite's folder is run by the browser runner. The type-checking half of the same constraint is in `editor-ui` U4. Both are invisible until they bite, and both bite in a way that points somewhere else. _[earned 2026-08-11]_

**One folder per layer, and the sorting rule is which project the test compiles in — not what it is about.** `tests/store/`, `tests/runtime/` and `tests/shell/` are browser-side and are added to `tsconfig.editor.json` and excluded from `tsconfig.json`; everything else stays in the Node project. The catch that is easy to get backwards: a test *about* browser-side code is not necessarily a browser-side test. The runtime-boundary check reads files with `node:fs`, so despite being entirely about `runtime/` it belongs in the Node project — it went to `tests/architecture/`. Sort by what the test imports, not by what it inspects. _[earned 2026-08-11]_

### V16: An injected clock and an injected writer are what make a store testable without waiting

The document store takes `now`, `saveDebounceMs` and `writeToDisk` as options; the app supplies the real ones, tests supply a clock they move by hand, a zero debounce, and a writer that records what it was asked to write and can be made to fail. **Reason:** the interesting properties are all about ordering — what merges into one undo step, what is written and in what order, what happens to a change while its write is in the air — and every one of them is either untestable or slow if the only way to cross a 600ms window is to wait 600ms. A store that has to be tested through the UI is a store whose ordering rules are not really tested at all. _[earned 2026-08-11]_

### V17: A canvas feature is verified by what the renderer reports and what the DOM shows — never by comparing pixels

Two halves, and neither is a screenshot baseline. The renderer reports what it actually did and the panel puts it on the element (`data-drawn-filter`, `data-drawn-version`, `data-scale`), so an assertion about the picture is an ordinary attribute assertion. Everything drawn *over* the canvas — frame guides, a pivot marker, a caption — is DOM (`editor-ui` U16), so it is counted and read with ordinary locators.

**Reason:** pixel comparison is brittle across machines and GPUs, and it is also the wrong instrument: the question is almost never "are these bytes identical", it is "is the renderer filtering this nearest". The load-bearing detail is that the reported value must be **read back off the renderer's own state, not echoed from the request** (`phaser4-runtime` P4) — an attribute built from what the panel asked for keeps asserting green after the line that applies it stops working, which is precisely the regression the test existed to catch.

Two assertions worth writing that this makes possible. That the *same canvas element* is still there after a setting changes, which is how "nothing was torn down and rebuilt" becomes checkable rather than aspirational. And that the drawn image's bounding box fits inside the stage's, which is a property of the zoom rather than a magic number. _[earned 2026-08-11]_

### V18: A value that settles asynchronously is read once it has stopped moving, not once it exists

A zoom test read the fit scale immediately after selecting a texture, got the fit for a panel that was about to get smaller, and failed later comparing against it. The helper polls until two consecutive reads agree, and returns that.

**Reason:** dockview computes its layout inside `requestAnimationFrame` (`editor-ui` UG1), so anything derived from a panel's size is briefly true and then differently true. A single read is not wrong, it is early — and the failure surfaces several lines away, in the assertion that compares against it, which reads as a bug in the feature. The same shape applies to any value a layout feeds. Tell for the sight test: a test that captures a number into a variable and asserts against it later, where the number depends on an element's size. _[earned 2026-08-11, dockview 8.0.0]_

### V19: A camera is asserted through what the renderer reports, and the load-bearing assertion is that framing is idempotent

A viewport camera is testable without a single pixel comparison, because the renderer reports the camera it drew with and the placement of everything it drew. That makes the acceptance criteria ordinary attribute reads: everything is in frame (count the entities whose reported rectangle intersects the canvas), a drag moved the level (the sprite, the crosshair and the origin marker each moved by the drag delta), a zoom is on the ladder (the reported scale is a whole number or the reciprocal of one), the level's units did not change (the Inspector's fields are unchanged across a zoom), and the view is not in the level (the scene file's bytes *and* mtime are unchanged after panning, per V12).

**The one that earns its place is pressing the frame key twice and asserting nothing changed.** Two independent defects both hide from every other assertion in the list and both surface to a human as a key that misbehaves: a level whose measured extent depends on the zoom it was measured at (`phaser4-runtime` P5), and a caption that appears only when something is off screen and so shrinks the canvas the framing is computed against (`editor-ui` UG8). Each produces a framing that is perfectly correct in isolation. Write the second press. _[earned 2026-08-12]_

### V20: A drag is asserted in the level's units, and the test that earns its place is the one at a different zoom

Placing a sprite by dragging it is testable without a pixel: the gesture is dispatched at the outline the renderer reported, and the result is read from the Inspector's fields and from the file on disk. That covers the ordinary claims — it moved, it landed on a whole unit, Alt put it between two, one press of Ctrl-Z took the whole drag back, a click did not nudge it, the position reached the file within the second.

**The one that catches a whole class of bug is the same drag repeated at a different zoom.** An implementation that never learned about the camera — one screen pixel treated as one level unit — passes every other assertion in the file, because they all happen at whatever zoom the scene opened at. Two practicalities for writing it: express the gesture as `units × scale` so the test says what it means, and **frame the entity before pressing on it at the new zoom**, because zooming about the middle of a level pushes anything near its edge off the panel and the press then lands on nothing. _[earned 2026-08-12]_

### V24: Two loaders of one document are checked by drawing both and comparing the reports, not by diffing the inputs

Play mode gave the kernel a second way to turn a level into a picture: the editor's incremental resolution, and the runtime's own loader. They must agree, and nothing about a disagreement announces itself — both halves produce something that looks like a level. So the editing view's report is kept at the instant Play is pressed and compared with the running level's, entity by entity: same ids, same screen origin, same bounds. The verdict is on the panel (`data-play-match`) and in a sentence a human reads.

**Compare the reports, not the two requests.** The request is the input and the report is the picture, and everything that could go wrong *between* them — a pivot applied differently, a frame cut differently, a texture that decoded on one side only — is invisible to an input diff. It is also cheap in exactly the way V17 describes: the renderer already reports what it drew, so this is arithmetic over two plain objects and one unit test file, with no pixels involved.

Four things that make it a real check rather than a tautology:

- **Both pictures must be through the same camera and the same canvas**, or every difference is real and none of them is about what is being checked. Mismatched camera or size is its own answer — "cannot be checked" — never a silent pass. The feature earns this by freezing the camera while a level runs.
- **The baseline must be a *settled* picture** (`editor-ui` U27), or the running level is being compared with a half-drawn one and the assertion is meaningless in the direction that looks green.
- **A tolerance, but a tiny one.** Both numbers come from one renderer, so agreement is exact; the tolerance exists only because the request makes a round trip through JSON. Assert both sides of it — that a thousandth of a pixel passes and a tenth fails — or the tolerance quietly becomes the thing being tested.
- **Name every difference, not a count.** A verdict that says "3 differences" is a verdict somebody has to reproduce by hand.

_[earned 2026-08-12]_

**Amended once the runtime had a clock: it is a check on the *first frame*, and the amendment is a pattern rather than a detail of this feature.** A running level moves and an editing view does not, so a check that kept comparing them would go red at the moment the product started working. Three parts, and the third generalises:

1. **The subject is the frame the level started on**, published under the name the hook already had (`data-scene-units`), with where the level has got to since beside it under a new one (`data-play-units`). Same on the exported side (`data-game-units` / `data-game-units-now`). **Do not repoint an existing hook at the moving value** — every test that compares two surfaces is written against the old name, and it would quietly start comparing two unrelated instants.
2. **The ordering is enforced, not timed.** The clock is refused permission to start until the renderer has drawn the running level and the baseline exists. Left to a race, the check compares against a level that has already moved by a fraction — which fails as a real-looking sub-pixel difference about as often as it passes.
3. **When a continuous check has to become a point-in-time one, ask what it was ever able to detect.** Here: a disagreement between two *loaders*, all of which is visible before anything moves. So the narrower check is worth what the old one was, and writing that down is what stops a later session "restoring" it.

**It has now caught something, and what it caught is worth recording because it was nowhere near it.** A session that added two number fields to the viewport's caption bar turned nine specs red across play mode and the export — all of them reporting `unavailable`, none of them about captions, controls or CSS. The fields had made the editing bar taller than the play bar, so the canvas resized at the moment Play was pressed and the two pictures were no longer comparable (`editor-ui` UG8 has the mechanism). **The refusal is what made it a failure rather than a slow drift**: had the comparison shrugged and compared two differently-sized canvases, it would have produced a handful of sub-pixel differences, which is exactly the shape of noise a session talks itself into tolerating. The design rule to carry: **"cannot be checked" must be a red test, never a quiet caveat** — its whole value is being unignorable in a file that knows nothing about the change that caused it. _[confirmed 2026-08-13]_

The narrowing needs a companion or it hides a regression: **a check that something does not move must be paired with one that something does.** There is now an assertion that after a quarter-turn exactly one entity's drawn rectangle changed — a runner that moves everything and a runner that moves nothing both fail it, and the second is precisely what "we drew the level once and called it running" looks like. _[earned 2026-08-13]_

### V25: "Nothing wrote to my file" is asserted by reading the bytes either side, with the destructive controls pressed in between

Play mode's promise is that a level can be run and stopped with the file untouched. The test snapshots every file the editor is able to write, plays, presses Add and Delete with `force` and Ctrl-Z, stops, waits past the autosave debounce, and asserts the whole snapshot is byte-identical.

Three parts, each load-bearing. **`force` on the clicks**, because an ordinary click on an `inert` control times out and proves nothing — the assertion is about what the press *did*, not about whether it was possible. **The wait past the debounce**, because a write that was scheduled and not yet flushed is exactly the failure being hunted, and a check that runs before the timer would miss it every time. And **the whole snapshot rather than the level**, because the interesting bug is a mode that writes somewhere nobody thought to look — which is also the file nobody would have listed. The same snapshot covers "nothing about play mode is recorded anywhere" for free.

**Reason:** it is the only formulation that tests the guarantee rather than the mechanism. Asserting that a store flag is set tests the flag; asserting the buttons are disabled tests the buttons. Neither would notice a third path to disk. _[earned 2026-08-12]_

**Extended when levels started moving, and the extension is what the test was always supposed to be about.** As written above it played a level that stood perfectly still — so it was a test of the *editor's controls* being out of reach, and it would have passed unchanged against a runner that wrote a hundred transforms a second into the human's file. It now plays a level whose entities genuinely move, waits until at least a hundred and twenty fixed steps have gone by, and only then presses anything. The claim and the test finally match: *the thing being mutated sixty times a second is a copy* (`editor-kernel` D27).

**And the snapshot now carries the modification time as well as the bytes** (V12, which said this from the beginning and was not being followed here). A write of identical contents leaves the bytes equal and the timestamp moved, and "the editor rewrote my level with exactly what was already in it" is still a promise broken — it churns git, and it is the shape this bug would most likely take, because the copy and the original start out identical.

The general form, worth more than either detail: **a test of "X cannot happen" is only as good as the run-up to it.** Ask what the system was actually doing while the test watched. If the answer is "nothing", the test is about the guard and not about the hazard. _[earned 2026-08-13]_

### V26: Two pictures that cannot share a canvas are compared in the subject's own units

V24 compares play mode with the editing view in screen pixels, and its first condition is that both came through the same camera onto the same canvas — mismatched size or camera is its own answer, never a silent pass. An exported game cannot satisfy that: it is a browser window and the editor's viewport is a docked panel, so the two frame the level at different zooms *by nature*, and every screen number differs while none of the differences is about what is being checked.

So the comparison moves into **the level's own units**, where the two agree exactly — a sprite's rectangle there is decided by its transform, its pivot and its frame size, and by nothing about the window it is seen through. The exported game and the editor's viewport each publish that projection, from **one shared function in the runtime** (`kernel-2d/runtime/scene/drawn-in-scene.ts`), and the test compares two plain lists.

Four things that make it a real check rather than a reformulation:

- **The projection must invert through the camera the renderer *drew* with, not the one it was asked for.** Pixel art is kept crisp by nudging the camera under a device pixel (`phaser4-runtime` P5), and the renderer reports the un-nudged camera on purpose. Inverting through that one is wrong by a fraction of a level unit — an eighth at 8× — which is small enough to read as noise and large enough to fail. So the report gained a second field for the camera actually used, and the test that catches this is the only one where the two differ; every other test in the file passes either way.
- **Assert that the two zooms are genuinely different.** If they ever coincided, the test would pass while checking something much weaker than it claims to. One line, and it is what stops this quietly degrading into a screen-pixel comparison.
- **Draw order is part of the picture.** Two levels holding the same entities in a different order overlap differently, and a comparison keyed by id alone calls them identical.
- **The instrument gets its own tests, including both edges of the tolerance.** A comparison that always answers "the same" passes every test written from the happy path. Each way two pictures can differ is a test, and so is "a thousandth of a unit passes, a tenth fails".

**Where it lives.** In `tests/`, because no product code compares itself to an editor — the same line `editor/shell/play-comparison.ts` is on, one layer out. Its home is `kernel-2d/tests/instruments/`, a folder for the suite's own tools, because a pure helper the browser suite imports has to compile in the Node project and its Vitest test cannot sit in the Playwright folder (V15). Instrument and test beside each other; the spec imports the instrument.

**It duplicates `play-comparison.ts`'s arithmetic on purpose, and a gardening pass has already confirmed it should stay that way** — the two compile in different TypeScript projects, the only home both could import is `runtime/`, and comparison-against-an-editor in the shipping layer is what D1 exists to prevent. Reasoning in full at `editor-kernel` G3; do not re-derive it. _[earned 2026-08-12, duplication confirmed correct 2026-08-12]_

### V27: An artefact is proved clean by absence, and absence needs two different instruments

"There is no editor in this folder" is `editor-kernel` D1 as something checkable, and it takes two checks because it fails in two unrelated ways.

- **A search for names that only exist on the far side of the boundary**, over the files the build *generated*. That is the check for editor code arriving through a path the import rule does not model — a bundler resolving something unexpected, a later session pointing the build at a different entry.
- **The folder listing has to equal the manifest of what was written.** That is the check for a *stray*: a file nobody bundled, which a name search cannot see because it does not know to look at it.

Three rules that came out of building it:

1. **Search only what was generated, never what was copied.** The copied files are the human's levels and import settings, byte for byte. A level whose entity happened to be called something on the list would be refused for a reason nobody could act on, and no quantity of a human's data is editor code.
2. **The command does the check, not only the suite.** The human asked to be told rather than left to find it, so a hit is a refusal from `npm run export` with the file and the marker named. The test asserts the same thing on a real folder — because a check that refuses is indistinguishable, from the outside, from a check that never ran.
3. **Assert that every marker still matches the thing it is looking for.** Tightening a marker into something unmatchable is a silent way to turn the whole check off, and it looks like a green suite.

**And prove the absence from the page as well as from the disk.** The exported page is asserted to have no assets panel, no inspector, no docking container, and to have made no request to the editor's service — watched with `page.on('request')` from outside the browser rather than from inside the page. That is the observable form of the claim, in the same vocabulary the human used. _[earned 2026-08-12]_

### V28: A move is proved by what did *not* change, and the assertion that carries it is the id

Renaming a file from the editor is testable without a pixel, and almost none of the assertions are about the screen.

**The one that carries the feature is that the reference's `id` is unchanged while its `path` is not.** A rename takes the `.meta` with the file, so the stable id at the new path is still the one every level recorded — an implementation that "fixed up" the id as well would draw the same picture, pass every screen assertion, and have quietly replaced the only thing that can notice a file being swapped. Assert the level's recorded id equals the id in the `.meta` at the *new* path, so the two are compared rather than each checked against a constant.

The rest of the file, in the order they earn their place:

- **Refusals first, each against bytes *and* timestamp** (V12) — and for a move there is a second half a create never had: **the source is still there too.** A move that half happened is a file destroyed rather than merely not moved.
- **The settings a human set survive**: write a tuned pivot and a grid slice, move the file, assert the whole `.meta` is equal. "The id survived" and "the settings survived" are the same mechanism and different promises.
- **The picture never breaks**: the level still draws every entity, and there is no problem sentence. Cheap, and it is the human's own acceptance criterion.
- **The neighbour that should not have moved** (V22): a document that referenced nothing is not rewritten at all.
- **The command downstream stops refusing.** The whole point of the fixup is that `npm run export` no longer names a picture it cannot find, so run the real command as a process after the rename (V8) and assert it does not refuse. It is the only assertion that checks the *reason the feature exists* rather than its mechanism.

**And the live-watcher test is not optional.** The ordering that keeps a rename's id — sidecar first, file second, with the service ignoring its own writes — can only fail against a real watcher, because there is nothing to lose the race to without one. A test against the functions alone passes with the rule removed entirely. Same shape as V2: the property under test belongs to the system, not to the module. _[earned 2026-08-12]_

## Gotchas

### W20: A forbidden-folder check that matches path segments cannot see a folder outside the repo, and reads as green

`tests/architecture/boundaries.test.ts` forbids the runtime importing from `editor/`, `sidecar/`, `scripts/` and `tests/` by resolving each specifier and taking the first segment of `path.relative(REPO_ROOT, resolved)`. Adding `games` to that list — the obvious way to assert "the kernel never imports from a sibling game folder" — produces a test that **can never fail**: for anything outside the repo the first segment is `..`, so the comparison against `'games'` is never true. It passes, it reads as a guard in the file, and nothing about it looks wrong.

Same family as W9 and V6 — a check whose green means nothing — but arriving by a different route. W9's mechanism is an empty input; here the input is full and the *predicate* is the thing that cannot fire.

**Fix/policy:** when the thing being forbidden lives outside the tree being scanned, assert the boundary as a **direction** rather than as a name — `path.relative(REPO_ROOT, resolved).startsWith('..')`, "no relative import escapes the repo". Three things fall out, and they are why this is the general answer rather than a workaround:

1. It catches the next sibling folder nobody has thought of yet, not just the one that prompted it.
2. It cannot go vacuous — it scans files that certainly exist, rather than needing the forbidden folder to be present.
3. It is layout-independent, so the repo cloned on its own still passes. A test naming `games/` would have needed a W9 guard asserting that folder exists, and would then go red for anyone who cloned the kernel without it.

And check the *scope* of the block you are extending while you are in there: the existing one scans `runtime/` only, because it is about what ships. A repo boundary is about all four layers. _[earned 2026-08-13, first genre spin-up]_

### W19: A fixed timestep whose size is not exactly representable makes `STEP * n` fewer than `n` steps

`STEP_MS` is `1000 / 60`, which is `16.666666666666668`. `STEP_MS * 3` is exactly `50` — **less** than three of those — so a test that hands the loop `STEP_MS * 3` and expects three steps gets two. The accumulator is right; the arithmetic that built the frame is what is wrong, and nothing about the failure says so. It reads as an off-by-one in the loop, which is the file you then go and stare at.

**Fix/policy:** a test that asserts an exact step count gives the loop **a step size of its own, in whole milliseconds** — `createLoop({ stepMs: 10 })` and a frame of `30`. That is not avoiding the real value out of squeamishness: the property being tested is "three steps' worth of time buys three steps", and expressing the frame as a multiple of an inexact constant tests the constant's binary representation instead. Keep `STEP_MS` for single frames (one of them is exactly one step, since it is the same value subtracted) and for assertions that are about ranges rather than counts.

The general form: **whenever a test builds an input by multiplying a constant the product also divides by, check the constant is exact first.** _[earned 2026-08-13]_

### W18: A test that acts on a file and then asserts on a panel has to wait for the *editor*, and "the file changed" is not that

W10 says: after touching a file the editor is watching, the next assertion must be about the editor. A rename made *by* the editor looks like it escapes that — the editor did it, so surely it knows — and it does not.

A spec renamed a texture, polled until the level file on disk held the new path, brought the Viewport tab forward and asserted the level still drew five entities. It drew four, for twenty seconds. The picture had not failed to recover: **selecting a texture brings the Texture tab forward (`editor-ui` U18), whether a file is a texture cannot be answered until the folder listing catches up a few hundred milliseconds later, and clicking Viewport inside that window means the Texture tab takes it straight back.** The assertion was then against a panel that had been unmounted mid-rename, frozen at whatever it last reported.

It cost an hour, and most of that was spent believing the product was broken — the failure says "4 entities drawn" and stays there, which is a far more alarming sentence than "you are looking at a stale attribute".

**Fix/policy:**

- **Wait for something the editor could only know if its folder listing had caught up** — the new path's row appearing in the Assets tree — and only then claim a tab. The file being right on disk proves the *service* did its job, which was never in doubt.
- **Two diagnostic moves that shortened this and would shorten the next one.** Give the failing assertion a much longer timeout first: "still wrong after twenty seconds" and "wrong for two hundred milliseconds" are different bugs and cost one run to tell apart. Then dump the panel's own sentence rather than an attribute — the caption named the file and said "not in the project folder", which pointed straight at the folder listing.
- **Do not reach for the embedded preview to debug a docking layout.** It is not compositing, so dockview never lays out and tab clicks do nothing (UG1/W4) — the same trap, one layer along, and it will happily eat the time the longer timeout just saved. _[earned 2026-08-12, dockview 8.0.0, Playwright 1.62.1]_

### W15: A "this must not appear" search over unminified output collides with the engine's own prose

The first real run of the export's editor-marker check refused a perfectly good folder, twice: `immer` matched the game engine describing a shader as "immersive", and `/api/` matched a documentation link ending `doc/api/jsts_geom_Triangle.js.html`. Both are the failure the check exists to prevent, pointed at the wrong thing — **which is worse than not checking, because it makes a working command look broken** and the natural next move is to loosen the check until it stops complaining.

The cause is structural rather than bad luck: the bundle is deliberately unminified so the identifiers are readable, and that means several thousand lines of the engine's comments are in the same haystack.

**Fix/policy, two rules that keep a list of forbidden strings honest:**

- **Match as a whole word wherever the marker's own edges are letters.** `\bimmer\b` does not match `immersive`, and no future one-word package name will either. Build the pattern from the string rather than hand-writing regexes per entry.
- **A marker whose edges are *not* letters has to be specific enough to survive prose.** `\b` between two non-word characters matches nothing, so applying it blindly to `/api/document` or `.tsx` would silently switch those markers off — the worst outcome available, since the check keeps passing. Spell such markers out until they cannot appear by accident: `/api/` became the three endpoints the editor actually uses.

Keep both false positives as tests. They are the cheapest possible regression guard and they document why the list looks the way it does. _[earned 2026-08-12]_

### W16: The browser suite run back-to-back exhausts the machine's sockets, and the failures look like product bugs

One full pass of the browser suite leaves roughly **3,800 sockets in `TIME_WAIT`** — a page load per test, and every module Vite serves. Windows holds those for a couple of minutes. Run the suite six or seven times in a row while hunting something and the machine runs out, at which point tests start failing in ways that point anywhere but the cause:

- `page.goto: net::ERR_NO_BUFFER_SPACE` in a `beforeEach`, which reads as the editor failing to start;
- `SyntaxError: Unexpected end of JSON input` from a probe that reads a file the editor is writing, which reads as a race in the product's autosave.

Two different specs failed on two consecutive runs, neither of them the spec under development, and neither reproduced in 108 targeted repeats of the specs that had just been written.

**Fix/policy:** before believing a browser-suite flake, count them — `netstat -ano | grep -c TIME_WAIT` — and check for stray listeners a killed run left behind (`netstat -ano | grep LISTENING`). If the count is in the thousands, wait for it to drain and run once more; that is the whole diagnosis. Kill any server started by hand for a manual check, because an orphan holding a port outlives the shell that started it.

The general shape, and the reason this is worth writing down rather than shrugging at: **a flake that appears only after several consecutive runs is evidence about the machine, not about the code**, and the instinct to harden whichever test happened to draw the short straw makes the suite worse. Ask what changed between the green runs and the red ones; if the answer is "nothing but the number of runs", the answer is the runs. _[earned 2026-08-12, Windows 11, Playwright 1.62.1]_

### W17: A recursive delete of a folder containing a directory junction can delete through it, and `node_modules` is the folder you will have linked

Setting up the parity drill means standing a second checkout beside the real one, and `node_modules` is the obvious thing not to duplicate — a junction (`New-Item -ItemType Junction`) is the cheap answer on Windows and is correct. Tearing the workspace down afterwards is where it goes wrong: `Remove-Item -Recurse -Force <workspace>` **followed the junction and emptied the real `node_modules`**, taking 62 packages down to 27 and leaving `.bin` empty.

It is a nasty failure for three reasons stacked up. `Test-Path node_modules` still answers **true**, because the folder is there and merely gutted. Nothing in git notices, because it is ignored. And the symptom arrives at the next command as `'tsc' is not recognized`, which reads as a broken PATH or a bad shell — anywhere but a delete that has already happened.

**Fix/policy, in order of preference:**

1. **Do not put the junction inside the folder you will recursively delete.** Point the workspace's config at the real `node_modules` instead, or keep the link outside the tree being removed. This removes the hazard rather than handling it.
2. If a junction must live there, **delete the link first and assert it is gone** before the recursive remove — `(Get-Item path -Force).Delete()` then a `Test-Path` check. Do not trust `-ErrorAction SilentlyContinue` around either step: the whole failure is a step that quietly did not happen.
3. **The repair is `npm ci`, and it is complete**, because the lockfile is the record. Five seconds, no judgement needed — which is worth knowing before spending any time diagnosing the PATH.

The general shape: **a link inside a scratch directory turns "delete the scratch directory" into an operation with a blast radius outside it**, and the blast lands in generated files that no version control is watching. Worth checking on sight whenever a temporary workspace is built by linking something expensive. _[earned 2026-08-12, Windows 11, PowerShell 5.1]_

### W13: A gesture is many events, so reading right after it reads the middle of it

A middle-drag test dispatched a move in eight steps and read the result as soon as anything had changed. It got seven eighths of the drag, and failed by exactly one step — which reads as the pan arithmetic being wrong rather than as the read being early. `await page.mouse.up()` resolves when the input has been dispatched, not when the application has finished responding to all of it.

**Fix/policy:** put the settle inside the gesture helper rather than at each call site, so no test can forget it — poll until the reported value stops moving (V18) as the last line of `dragBy`, `wheelAt` and anything else that produces a stream of events. This is V18's rule with a different cause: there the value was settling because a *layout* was, here because the gesture was still arriving. Same instrument, and the same tell — a number captured into a variable and compared later. _[earned 2026-08-12, Playwright 1.62.1]_

### W8: Clicking a control and reading the result in the next statement reads the value from before the click

`await button.click()` resolves when the click has been dispatched, not when React has re-rendered in response to it. `const after = await scaleOnScreen(page)` on the following line is a coin toss, and it lands on "unchanged" often enough to look like the button is broken. **Fix/policy:** put a poll between the act and the read — `await expect.poll(read).not.toBe(before)`, then read. This is not the same as V5's no-fixed-sleeps rule, which is about waiting for the *system*; this is about waiting for the *framework*, and it applies to every assertion that follows a click on a control whose effect is state rather than navigation. _[earned 2026-08-11, Playwright 1.62.1, React 19.2.8]_

### W9: A test that scans a folder passes vacuously when the folder is gone

The runtime-boundary test walks `runtime/` and asserts no file imports from `editor/`. Renaming or moving that folder makes the walk return nothing, every assertion trivially true, and the suite green while the check it represents has stopped existing. **Fix/policy:** any test that derives its cases from a directory listing asserts the listing is non-empty first, as its own named test. One line, and it is the difference between a guard and a decoration. Same shape as V6's anchoring rule: an absence only means something once something else has proved the mechanism was live. _[earned 2026-08-11]_

### W7: A test can be green *because* of the bug, and fixing the bug is what reveals it

An Inspector test asserted that selecting a README says "the editor does not import this kind of file". It passed for months of nothing and failed the moment a one-render staleness bug was fixed (`editor-ui` UG5) — because what it had actually been catching was the render in which the *previously selected folder's* answer was shown under the README's name. The sentence it asserted was real; it just belonged to a different file. The file it was pointed at has a `.meta` and says something else entirely.

**Fix/policy:** when a test fails immediately after a fix that "should not have touched it", check whether the fix removed the thing the test was really observing before assuming the fix is wrong. The tell is a test that passes while asserting a state the code under test cannot actually be in for the input given — worth spending five minutes confirming the state is reachable at all. The repair is usually two tests: one for what the file actually does, and one that constructs the input the original sentence really describes. _[earned 2026-08-11]_

### W5: `waitFor` takes a synchronous probe, and an `async` one silently makes the wait a no-op

`waitFor(async () => (await exists(path)) ? true : undefined, …)` returns a *promise* on every tick. A promise is never `undefined`, so the very first poll "succeeds", the wait returns immediately, and every assertion after it runs against a folder nothing has happened in yet. The test passes, proves nothing, and looks like the most convincing test in the file. **Fix/policy:** probes are synchronous — `existsSync`, a captured array's length, a locator count already resolved. If a probe genuinely must await something, the helper needs to await it, and that is a change to the helper rather than to one call site. Worth checking on sight in any new suite: an `async` arrow passed to a poller is a defect until proven otherwise. _[earned 2026-08-11]_

### W6: A helper that clicks a folder open closes it the second time a test calls it

A `select(path)` helper that clicks every ancestor on the way down works perfectly in the first test that uses it and silently collapses the tree in the second, because clicking a folder is a *toggle*. The failure surfaces as "element not found" three lines later, which reads like a rendering bug. **Fix/policy:** helpers that navigate a tree check the state before acting — read `aria-expanded` and click only when it is shut — so calling one twice is the same as calling it once. Any test helper that drives a toggle needs the same treatment. _[earned 2026-08-11, Playwright 1.62.1]_

### W1: Asserting on the exact shape of formatted output tests the formatter's accidents

An assertion that the JSON response contained `"\n      \"kind\": \"directory\""` failed purely because a node sat one nesting level deeper than guessed — the response was correct. **Fix/policy:** assert the observable property that was actually promised ("it is indented and multi-line, so a browser can read it raw") rather than a byte-exact rendering. The same applies to terminal output: assert that the columns line up, not that a line equals one exact string, unless the exact string *is* the contract. _[earned 2026-08-11]_

### W2: Watcher teardown must be ordered, not just present

The suite closes every watcher before deleting its temp folder, in `afterEach` for shared watchers and in a `finally` for tests that make their own. This ordering was written in from the start rather than discovered, so treat it as a precaution rather than a scar: on Windows a held handle can block folder removal, and the resulting failure surfaces in cleanup rather than in the test that caused it. If a future session ever sees a cleanup-time permissions error, this is the first thing to check. _[earned 2026-08-11 — precaution, not yet observed failing]_

### W3: Dockview's tab drag is HTML5 drag-and-drop, which hand-rolled mouse steps do not trigger

A careful `mouse.down` → several `mouse.move`s → `mouse.up` sequence moves the pointer and does nothing else: no `dragstart`, no drop, and a test that fails while the same gesture works perfectly by hand. **Fix/policy:** use Playwright's `locator.dragTo()`, which drives Chromium's drag pipeline properly. Reach for manual mouse steps only for gestures that really are pointer-driven — a splitter drag, for instance, which is. _[earned 2026-08-11, Playwright 1.62.1, dockview 8.0.0]_

### W4: A browser surface that is not painting produces layout failures that are not real

Anything sized through `requestAnimationFrame` — dockview's whole grid, among others — stays at its initial dimensions in a preview pane that is not compositing, so panels measure 100px and splitters report themselves disabled. Headless Chromium under Playwright *does* paint, so the suite is trustworthy where an embedded preview is not. **Fix/policy:** when an embedded browser shows a collapsed layout, reproduce it in the Playwright run before believing it. See `editor-ui` UG1. _[earned 2026-08-11]_

### W10: A test that drives the editor after hand-editing a file must wait for the *editor* to have taken it, not for the file to be on disk

A test wrote extra keys into a scene, polled until the file contained them, then typed into a field and asserted the keys survived. The poll only ever proved the test's own write had landed — the editor had not re-read yet, so the keystroke was applied to the document it was still holding and the write put the pre-edit version straight back over the keys. The test failed for a reason that reads exactly like the feature being broken.

**Fix/policy:** wait for something *on screen* that could only be true if the editor had taken the file. Make the hand-edit change something visible — rename an entity, add one — and wait for that, then act. The general rule: after touching a file the editor is watching, the next assertion must be about the editor, never about the file. Same shape as V6's anchoring rule, applied to a round trip rather than to an event. _[earned 2026-08-11]_

### W11: An attribute used as a test hook has to be unique across the whole page, not within the panel that owns it

The Hierarchy's rows carried `data-entity-id`, and so did the SVG group that outlines the selected entity in the viewport — so `[data-entity-id]` counted one more row than existed, and a list of names came back with an empty string on the end. Both readings are reasonable in the file that wrote them; together they are wrong, and the failure looks like an off-by-one in the panel.

**Fix/policy:** two ways out, and use both. Name the attribute for what it *is* rather than for what it contains, so the overlay says `data-selected-entity` and cannot collide; and scope the locator to the panel that owns it. Worth checking on sight whenever a second panel starts describing the same objects: the moment two panels talk about entities, their test hooks are in the same namespace. _[earned 2026-08-11, Playwright 1.62.1]_

### W14: The shared restore only covers files the *editor* can write, so a test that removes anything else must put it back itself

V14's snapshot-and-restore walks the sample folder for the extensions the editor is allowed to write — `.meta` and `.json`. A test that deletes a PNG to see what a missing texture does therefore deletes it **for the rest of the run**, and the suite goes red in whichever later spec happens to open a level that used it. It passes on its own, passes in its own file, and fails only in a full run, pointing at code that has nothing to do with the cause.

**Fix/policy:** read the bytes, delete, assert, write them back in a `finally`. Do not widen the shared restore to cover binaries instead — that list is the statement of what the editor is permitted to write, and quietly turning it into "everything a test might touch" costs the reader the one place that answer is written down. _[earned 2026-08-12]_

**Sharpened once the editor could move files: a restore that puts files back *by path* cannot survive a folder that moved.** The shared snapshot recreates each `.meta` and `.json` at the path it had, and `writeFileSync` throws when the folder it names is no longer there. So a test that renames a folder must **rename it back**, in a `finally`, before the shared restore runs — which also brings every binary inside it back for free. The tell to watch for: a shared restore that assumes the *shape* of the folder is fixed, in a suite that has just gained the ability to change it. _[earned 2026-08-12]_

### W12: A tab sharing a dockview group is not on screen, so a spec must bring it forward before asserting anything

The Texture tab sits behind the Viewport in the same group. Every assertion about it — the canvas being visible, a caption, an overlay — is against a panel that is not rendering until something activates it. Selecting a texture does that in the product; a spec asserting the empty state *before* selecting anything has to click the tab itself.

**Fix/policy:** a shared `showPanel(page, title)` helper that clicks the tab and waits for `aria-selected`, called in `beforeEach` for any spec whose subject shares a group. Do not reach for `{ force: true }` or for asserting against hidden elements — the panel really is not there, and a test that pretends otherwise is asserting against something the human cannot see. _[earned 2026-08-11, dockview 8.0.0]_

## Contracts

- `kernel-2d/tests/fixtures/project-fixture.ts` — the temp-project builder, `waitFor`, and `delay`. Everything filesystem-shaped in the suite starts here.
- `kernel-2d/tests/sidecar/watcher.test.ts` — the worked example of V1, V2, V5, and V6.
- `kernel-2d/tests/sidecar/tree-schema.test.ts` — the round-trip pattern of V7.
- `kernel-2d/tests/sidecar/status-schema.test.ts` — V7 again, for a format that also crosses the wire: the same object is checked through memory and through the served response.
- `kernel-2d/vitest.config.ts` — suite configuration, including the timeout the real-filesystem tests need.
- `kernel-2d/playwright.config.ts` — the browser harness: how the editor is started, on which ports, against which throwaway project (V8).
- `kernel-2d/tests/editor/shell.spec.ts` — the browser suite as it stands, and the worked examples of V9, W3, and the screenshot habit below.
- `kernel-2d/tests/editor/assets.spec.ts` — V11, and the panel's behaviour against the real sample folder.
- `kernel-2d/tests/editor/inspector.spec.ts` — every state a panel can be in, asserted as the sentence the human reads.
- `kernel-2d/tests/editor/select-asset.ts` — the tree-navigation helper of W6, shared by every spec that needs something selected.
- `kernel-2d/tests/editor/import-settings.spec.ts` — the acceptance criteria for editing, transcribed in the human's units (V1), plus the snapshot-and-restore of V14.
- `kernel-2d/tests/store/documents.test.ts` — the transaction API on its own (V15, V16): one stack across two files, what merges into one undo step, and every way a re-read can be stale.
- `kernel-2d/tests/editor/texture.spec.ts` — V17 and V18: a canvas feature asserted without comparing a single pixel, and the settle helper that keeps a layout-derived number honest.
- `kernel-2d/tests/editor/scene.spec.ts` — the same instruments one layer up: where an entity was drawn, whether a feet-pivoted sprite stands on its position, and a missing texture named rather than drawn as nothing.
- `kernel-2d/tests/editor/hierarchy.spec.ts` — add, delete and reorder, each of them taken back by one press of Ctrl-Z. The behavioural proof that document-level undo covers a tool nobody wrote undo for.
- `kernel-2d/tests/editor/restore-project.ts` — V14 for every file the editor can write, shared by every spec that changes the sample folder.
- `kernel-2d/tests/editor/panels.ts` — W12: bringing a tab forward before asserting against it.
- `kernel-2d/tests/runtime/scene-schema.test.ts` and `kernel-2d/tests/runtime/scene-coordinates.test.ts` — V7 for the scene format, the y-up arithmetic under the pivot assertion, and the camera held to the four properties its acceptance criteria are made of.
- `kernel-2d/tests/editor/scene-camera.spec.ts` — V19: a camera asserted without a pixel, one test per acceptance criterion, and the settled gesture helpers of W13.
- `kernel-2d/tests/editor/drag-place.spec.ts` — V20: picking and placing asserted in the level's own units, with the zoom-invariance test that catches a camera-blind implementation.
- `kernel-2d/tests/editor/snap-place.spec.ts` — V20 again for a settable grid and a mode that places on every click: the offset test that decides whether the snap is worth having, the three-clicks-without-touching-anything-else test of `editor-ui` U31, and the click-in-the-middle-of-the-canvas test that is this feature's camera-blindness catcher.
- `kernel-2d/tests/shell/snap.test.ts` — the same arithmetic as properties: a sloppy row of clicks landing exactly one step apart, idempotence, and what actually reaches the file.
- `kernel-2d/tests/shell/drawn-entities.test.ts` — what can be clicked, held to being the same rectangle the outline is drawn from.
- `kernel-2d/tests/editor/new-scene.spec.ts` — V21: making a level, and every way it is refused.
- `kernel-2d/tests/editor/prefabs.spec.ts` — V22: the two assertions a copying implementation cannot pass — the level holding no copy of what it inherits, and the level's bytes unchanged by an edit to the prefab.
- `kernel-2d/tests/sidecar/document-endpoint.test.ts` — the create and the replace held to refusing each other's job, refusals first, and neither format allowed to be written over the other.
- `kernel-2d/tests/runtime/frames.test.ts` — the frame geometry, held to "every frame reported is a whole frame, and every pixel outside one is counted".
- `kernel-2d/tests/shell/zoom.test.ts` — the zoom ladder held to the one property that matters: every step is whole.
- `kernel-2d/tests/architecture/boundaries.test.ts` — W9, and two boundaries held as tests rather than promises: what ships (runtime never imports the editor, `editor-kernel` D20) and where the repo ends (no relative import escapes it, W20).
- `kernel-2d/tests/sidecar/asset-endpoint.test.ts` — the read privilege of `editor-kernel` D21 held to its four lines, refusals first.
- `kernel-2d/tests/runtime/load-scene.test.ts` — the runtime's loader against a fake `ProjectReader`: no browser, no service, no folder, no renderer — and the only place the runtime's own validation is exercised at all (`editor-kernel` D26, point 2).
- `kernel-2d/tests/shell/play-comparison.test.ts` — V24 as arithmetic: what counts as the same picture, and each way two pictures can differ named in the human's units.
- `kernel-2d/tests/editor/play.spec.ts` — V24 and V25 end to end, plus W14: a level run from the file, checked against the editing view, stopped, and the folder proved untouched.
- `kernel-2d/tests/sidecar/meta-write.test.ts` — the service's editor-driven write held to its edges, with every refusal checked against bytes *and* timestamp (V12).
- `kernel-2d/tests/sidecar/meta-schema.test.ts` — V7 for the `.meta` format, and the rejections that make it a contract.
- `kernel-2d/tests/sidecar/meta-files.test.ts` — V12: the sidecar's write privilege held to exactly its three lines, against a real filesystem.
- `kernel-2d/tests/sidecar/meta-generation.test.ts` — the same rules through the running service (V13), including the "within a second" budget and the assertion that the folder settles rather than feeding itself.
- `kernel-2d/tests/sidecar/meta-endpoint.test.ts` — the answer the Inspector receives over the wire, and every path-escape attempt refused.
- `kernel-2d/tests/editor/test-project.ts` — the throwaway project the browser tests point the editor at, built at run time under `tests/.tmp/` and git-ignored (V10).
- `kernel-2d/tests/sidecar/events.test.ts` — the change feed end to end: real folder, real watcher, real HTTP stream, including the reader that parses server-sent frames and the ordered teardown a live stream needs.
- `kernel-2d/tests/scripts/sample-project.test.ts` — how a content generator is held to its promises: real file formats, identical bytes on every run, and never touching an unmarked file.
- `kernel-2d/tests/scripts/export.test.ts` — a command that produces an artefact, held to its refusals first: ten of them, none needing a build, plus the four tests that do build — including "export twice, byte-identical folders".
- `kernel-2d/tests/instruments/drawn-comparison.ts` and its test — V26: two pictures compared in the level's own units, and each way they can differ named.
- `kernel-2d/tests/runtime/drawn-in-scene.test.ts` — the projection V26 rests on, including the one test that distinguishes the drawn camera from the requested one.
- `kernel-2d/tests/editor/export.spec.ts` — V26 and V27 end to end: a served folder that plays the game, checked against play mode, moved somewhere else, opened off the disk, and proved to hold no editor.
- `kernel-2d/tests/editor/project-settings.spec.ts` — choosing the starting level, in the human's units: it reaches the file, survives a reload, refuses a prefab, and Ctrl-Z takes it back.
- `kernel-2d/tests/sidecar/file-operations.test.ts` — V28's refusals half: the move and the delete held to their edges against a real filesystem, every refusal checked against bytes, timestamp *and* the source still being there.
- `kernel-2d/tests/sidecar/meta-generation.test.ts` — where the ordering rule is actually provable: a rename through the running service, against a live watcher, keeping its id.
- `kernel-2d/tests/editor/rename.spec.ts` — V28 end to end, W14's binary restore, and W18's wait-for-the-editor rule; ends by running the real export command to show the refusal this feature exists to remove is gone.
- `kernel-2d/tests/shell/references.test.ts` — the rewrite as arithmetic: the id untouched, a folder prefix, a hand-added key surviving, and a document that referenced nothing coming back as nothing.
- `kernel-2d/tests/editor/test-export.ts` — V8 extended: the exported game the browser suite opens, built by running the real command.

### V21: A create is tested from the side where it refuses, and a picture waits for everything it is a picture of

Two things learned adding the first feature that puts a file into a project folder.

**The interesting half of "it makes a file" is "it will not make this one."** The happy path is one assertion; the guard that stands between the feature and somebody's finished level is that asking for a name already taken changes nothing — checked against the existing file's bytes *and* its timestamp (V12), because identical contents alone would also pass for a file that had been rewritten with the same text. Same for the folder it declined to create: assert the folder is *not there*, not merely that the file is not. And since creating and replacing are two separate requests (`editor-kernel` D22), each is tested for refusing to do the other's job — that pair is the whole safety argument, and testing one side of it proves half of nothing.

**A screenshot must wait for every panel in it, not just the first one to settle.** The picture of a newly-made level was taken as soon as the scene opened, and caught the Inspector saying the file was not in the project — true of the folder listing it was holding, which arrives a beat later by way of the watcher. The image looked like a bug that did not exist. Wait for the slowest thing in frame; a screenshot that documents a transient is worse than no screenshot, because it is the artefact somebody reaches for when they are already suspicious. _[earned 2026-08-12]_

### V22: When a feature is about a link, assert what the file does *not* contain — and that the file nobody edited is byte-identical

Prefabs are a promise about two files at once: the level holds a reference and no copy, and editing the prefab reaches every instance. Both halves are invisible to the assertions a session naturally reaches for, because a design that copied the picture into each instance at placement time would draw the same picture, list the same rows, and pass every test written from the screen.

Two assertions catch it and nothing else does:

- **Read the level and assert the entity's component map is exactly `['prefab']`.** A positive assertion ("it draws the slime") is satisfied by the broken design too. The absence is the property.
- **Fingerprint the file the human did not touch, change the prefab, and assert the fingerprint is unchanged** — bytes *and* timestamp (V12). An editor that copied would have had to rewrite the level, so this is the one assertion the wrong design cannot pass.

Generalises past prefabs: whenever a feature's whole value is *not duplicating something*, the test has to be about the thing that should not be there, and about the neighbour that should not have moved. Write both early in the file, where they will not be dropped for being awkward to phrase. _[earned 2026-08-12]_

### V23: A guard that is working makes an unrelated test fail somewhere it is not mentioned

Three prefab tests failed with the service refusing to create the file — correctly, because it never makes a folder on the way (`editor-kernel` D22) and the temp project had no `prefabs/` in it. The failures read as "creating a prefab is broken", and the first instinct was to suspect the new feature.

**Fix/policy:** when a suite tests writing into a folder, put the empty folder in the fixture (`'prefabs/': ''`, per V3) and name in a comment which guard would otherwise fire. The general shape is worth recognising on sight — same family as W7 and W9: a test whose failure message describes the wrong subject. Before changing the code under test, check whether some *other* rule is behaving exactly as designed. _[earned 2026-08-12]_

**The screenshot habit.** A test that asserts nothing and simply writes a full-window screenshot to its output path. It is not a visual-regression baseline — those are brittle across machines — it is a picture to look at when something is reported as looking wrong, which beats reasoning about the source.

**One per surface that can look wrong on its own, and no more.** That is three now: the shell, the texture tab and the scene. The original rule said "exactly one", and what matters is the reason it moved rather than the number — a second canvas panel is a genuinely different picture, and reasoning about it from the shell's screenshot is the guessing this habit exists to avoid. A screenshot per *test* is still noise, and a screenshot of something with no picture in it always was.
