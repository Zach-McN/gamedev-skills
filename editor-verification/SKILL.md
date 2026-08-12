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

## Gotchas

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
- `kernel-2d/tests/runtime/frames.test.ts` — the frame geometry, held to "every frame reported is a whole frame, and every pixel outside one is counted".
- `kernel-2d/tests/shell/zoom.test.ts` — the zoom ladder held to the one property that matters: every step is whole.
- `kernel-2d/tests/architecture/boundaries.test.ts` — W9, and the runtime/editor boundary as a test rather than a promise (`editor-kernel` D20).
- `kernel-2d/tests/sidecar/asset-endpoint.test.ts` — the read privilege of `editor-kernel` D21 held to its four lines, refusals first.
- `kernel-2d/tests/sidecar/meta-write.test.ts` — the service's editor-driven write held to its edges, with every refusal checked against bytes *and* timestamp (V12).
- `kernel-2d/tests/sidecar/meta-schema.test.ts` — V7 for the `.meta` format, and the rejections that make it a contract.
- `kernel-2d/tests/sidecar/meta-files.test.ts` — V12: the sidecar's write privilege held to exactly its three lines, against a real filesystem.
- `kernel-2d/tests/sidecar/meta-generation.test.ts` — the same rules through the running service (V13), including the "within a second" budget and the assertion that the folder settles rather than feeding itself.
- `kernel-2d/tests/sidecar/meta-endpoint.test.ts` — the answer the Inspector receives over the wire, and every path-escape attempt refused.
- `kernel-2d/tests/editor/test-project.ts` — the throwaway project the browser tests point the editor at, built at run time under `tests/.tmp/` and git-ignored (V10).
- `kernel-2d/tests/sidecar/events.test.ts` — the change feed end to end: real folder, real watcher, real HTTP stream, including the reader that parses server-sent frames and the ordered teardown a live stream needs.
- `kernel-2d/tests/scripts/sample-project.test.ts` — how a content generator is held to its promises: real file formats, identical bytes on every run, and never touching an unmarked file.

**The screenshot habit.** A test that asserts nothing and simply writes a full-window screenshot to its output path. It is not a visual-regression baseline — those are brittle across machines — it is a picture to look at when something is reported as looking wrong, which beats reasoning about the source.

**One per surface that can look wrong on its own, and no more.** That is three now: the shell, the texture tab and the scene. The original rule said "exactly one", and what matters is the reason it moved rather than the number — a second canvas panel is a genuinely different picture, and reasoning about it from the shell's screenshot is the guessing this habit exists to avoid. A screenshot per *test* is still noise, and a screenshot of something with no picture in it always was.
