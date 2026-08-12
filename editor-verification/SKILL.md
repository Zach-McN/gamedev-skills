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

### V16: An injected clock and an injected writer are what make a store testable without waiting

The document store takes `now`, `saveDebounceMs` and `writeToDisk` as options; the app supplies the real ones, tests supply a clock they move by hand, a zero debounce, and a writer that records what it was asked to write and can be made to fail. **Reason:** the interesting properties are all about ordering — what merges into one undo step, what is written and in what order, what happens to a change while its write is in the air — and every one of them is either untestable or slow if the only way to cross a 600ms window is to wait 600ms. A store that has to be tested through the UI is a store whose ordering rules are not really tested at all. _[earned 2026-08-11]_

## Gotchas

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
- `kernel-2d/tests/sidecar/meta-write.test.ts` — the service's editor-driven write held to its edges, with every refusal checked against bytes *and* timestamp (V12).
- `kernel-2d/tests/sidecar/meta-schema.test.ts` — V7 for the `.meta` format, and the rejections that make it a contract.
- `kernel-2d/tests/sidecar/meta-files.test.ts` — V12: the sidecar's write privilege held to exactly its three lines, against a real filesystem.
- `kernel-2d/tests/sidecar/meta-generation.test.ts` — the same rules through the running service (V13), including the "within a second" budget and the assertion that the folder settles rather than feeding itself.
- `kernel-2d/tests/sidecar/meta-endpoint.test.ts` — the answer the Inspector receives over the wire, and every path-escape attempt refused.
- `kernel-2d/tests/editor/test-project.ts` — the throwaway project the browser tests point the editor at, built at run time under `tests/.tmp/` and git-ignored (V10).
- `kernel-2d/tests/sidecar/events.test.ts` — the change feed end to end: real folder, real watcher, real HTTP stream, including the reader that parses server-sent frames and the ordered teardown a live stream needs.
- `kernel-2d/tests/scripts/sample-project.test.ts` — how a content generator is held to its promises: real file formats, identical bytes on every run, and never touching an unmarked file.

**The screenshot habit.** One test in the browser suite asserts nothing and simply writes a full-window screenshot to its output path. It is not a visual-regression baseline — those are brittle across machines — it is a picture to look at when something is reported as looking wrong, which beats reasoning about the source. Keep exactly one; a suite full of them is noise.
