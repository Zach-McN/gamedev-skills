---
name: editor-ui
description: React, docking, and inspector idioms for game editors — the established patterns for building the editor's panels, layouts, property inspectors, and control surfaces. Consult this skill whenever building or modifying editor UI, laying out docked panels, wiring an inspector to the document model, or deciding how a React component in the editor should be structured.
---

# editor-ui

The idiom guide for the editor's user interface — how the React layer, docking system, and property inspectors are built and wired to the document model. This skill captures the reusable UI patterns the editor depends on so panels, inspectors, and controls stay consistent and predictable across the tool. Content is earned from real UI-building sessions, never invented.

Targets React 19, Vite 8, and dockview-react 8.

## Decisions

### U1: Panels are declared once, and that declaration produces both the components and the opening layout

One file holds every panel's id, tab title, and description; the dockview component registry and the startup layout function are both derived from it, and the panel id doubles as the dockview component name. **Reason:** the alternative is a component map and a layout script that each list the panels, which drift the first time someone adds a panel to one and not the other — and the symptom is a blank panel body, which reads as a rendering bug rather than a registration one. _[earned 2026-08-11]_

### U2: The editor never knows the sidecar's port — it asks its own origin, and the dev server proxies

Browser code fetches `/api/…`; Vite proxies `/api` to the sidecar and strips the prefix. The launcher tells Vite which port it picked through the environment. **Reason:** one origin means no CORS handling and no port literal compiled into the UI, so the sidecar can move ports (a busy default, a second project open, a test harness) without touching editor code. It also keeps the eventual Tauri swap (`editor-kernel` D10) confined to one layer. _[earned 2026-08-11]_

### U3: The shell always shows which project it is connected to, and admits when it is connected to nothing

A status strip carries the project's short name, its full resolved path, and a live connection state; the window title carries the project name too. The state is polled — briskly while disconnected, slowly while connected — rather than fetched once. **Reason:** the failure this prevents is a stale project name sitting on screen while the sidecar is dead, which invites the human to author into a folder nothing is watching. The full path is spelled out, not abbreviated, because two projects with the same folder name is ordinary. _[earned 2026-08-11]_

### U4: The browser half and the Node half are separate TypeScript projects, with different import conventions

`tsconfig.json` covers `sidecar/`, `scripts/`, `tests/` and the build configs (NodeNext, explicit `.js` extensions on relative imports); `tsconfig.editor.json` covers `editor/` (bundler resolution, DOM lib, `react-jsx`, extensionless relative imports); both extend one `tsconfig.base.json` holding the strict flags. `npm run typecheck` runs both. **Reason:** no single config serves both — NodeNext demands the `.js` extension that bundler-resolution code should not carry, and the DOM lib has no business in the sidecar. Files shared across the boundary (schemas) are imported with each side's own convention. _[earned 2026-08-11]_

**Amended: a unit test of browser-side code belongs to the browser project, and cannot live in the browser *suite's* folder.** Vitest tests for editor modules go in `tests/store/`, added to `tsconfig.editor.json` and excluded from `tsconfig.json`. Two separate constraints force this and neither is obvious: the Node project's NodeNext resolution cannot follow the editor's extensionless relative imports, so the test has to be type-checked with the editor's config; and Playwright's default match claims `*.test.ts` as well as `*.spec.ts`, so a Vitest file placed in `tests/editor/` is picked up by the browser runner and fails in a way that looks nothing like the cause. _[earned 2026-08-11]_

### U5: A panel that changes re-reads the whole thing rather than patching its copy

When the sidecar reports a change, the Assets panel fetches the entire folder again instead of applying the change to the tree it already holds. **Reason:** patching means a second piece of code that decides what the folder contains, and the two eventually disagree in a way nothing reports — the same failure as serialization drift, one layer up. A full re-read is one reader, always right by construction, and a scan of a game project is cheap next to the second the human is willing to wait. Revisit only when a project is big enough that the read is measurably late, and then measure rather than assume. _[earned 2026-08-11]_

### U6: Bursts are settled before acting on them, and the timing budget is stated where it is spent

Change events arrive in bursts — copying a folder in fires one per file — so the re-read is debounced by 75ms. That number is spent out of the human's one-second budget, alongside the watcher's 200ms write-settling delay (`editor-kernel` G6), and both are written down where they are chosen. **Reason:** the budget is the acceptance criterion, and a session that spends 300ms of it without noticing is how a sub-second requirement quietly becomes a two-second one. _[earned 2026-08-11]_

### U7: A panel that has lost its feed keeps showing what it has, labelled

When the change stream drops, the Assets panel keeps the tree on screen and says it may be out of date, rather than blanking or silently pretending it is current. **Reason:** stale-but-labelled is more useful than empty, and both are far better than stale-and-confident. Same reasoning as U3, and the two states are distinct: `unavailable` means never got a folder at all, `ready` with the feed down means this was true a moment ago. _[earned 2026-08-11]_

### U8: Selection is UI state, lives outside the document, and is never undoable

What is selected is held in a plain React context above the docking layout, not in the document store. **Reason:** pressing undo after clicking a different file must reverse the last thing that was *changed*, not the last thing that was *looked at* — and the way to guarantee that is for selection never to be in the thing undo replays. It is also never serialized and never appears in a saved file, which is the other half of the same observation. A context rather than a store because a single path needs nothing more; when the store arrives for real document state, selection stays out of it. _[earned 2026-08-11]_

**Amended when an entity became selectable: selection is a union, and what is *open* is a second thing that is not selection at all.**

```ts
type Selected =
  | { kind: 'none' }
  | { kind: 'file';   path: string }
  | { kind: 'entity'; scene: string; entity: string }
```

Two changes, each with its own reason.

**A union rather than a path plus an entity id.** A pair can spell states that are not real — an entity id belonging to a scene that is not the selected file, an entity id while a texture is selected. Four combinations, three of them meaningful, and every reader has to decide for itself which to ignore; they will not all decide the same way. The union cannot write the impossible ones down, so no reader has to guard against them.

**Which scene is open is separate state, sitting beside selection and equally never undoable.** With a level open, clicking a PNG means two things are being looked at and only one of them is selected. Folding "open" into selection would mean that click either closed the level or left a scene path in a field that also means "what is selected" — one field with two meanings. Keeping them apart is what makes "the Viewport still holds my scene while I look at a texture" fall out rather than be arranged.

The rule that follows and is worth stating on its own: **nothing decides a file is a scene from its path.** Selecting a `.json` is what makes the editor *read* it; the `format` inside it is what makes it the open scene (U11). A `scenes/` folder is a convention in the folder map, not a fact the code may rely on. _[earned 2026-08-11]_

### U9: Anything with a live subscription is read once per window and shared, not per panel

The Assets panel and the Inspector both need the project folder. The hook that fetches it and holds a change stream open is called once, in a provider above the layout, and both panels read from that. **Reason:** two callers means two streams, two fetches, and — the part that actually bites — two copies refreshed on separate timers, so one panel can be a beat behind its neighbour with nothing on screen saying so. Same reasoning as U5 one level up: the failure is not the wasted work, it is the two answers. **Providers go above the docking layout**, because dockview mounts and unmounts panel bodies as tabs move, and state held inside a panel is lost the first time the human drags it. _[earned 2026-08-11]_

**Extended twice, and the second reason is stronger than the first.** The selected file's `.meta` moved up here the moment the Viewport wanted it as well as the Inspector. The U9 reason applies as stated — but there is a second one that only shows up when the fetch has a side effect. Asking for the settings is also what *puts them in the document store*, so with the fetch owned by the Inspector, closing that tab would leave the Viewport drawing a texture whose settings never arrived: a panel silently depending on another panel being open, with no error and nothing on screen to explain it. **A fetch that populates shared state is not a panel's to own, whatever the caller count.** Worth checking on sight: if a hook writes somewhere other than its own component, it belongs above the layout.

**A live renderer belongs up here too, and for a harder reason.** A Phaser game owned by the Viewport panel is destroyed and rebuilt the first time somebody drags that tab — the same teardown a per-selection design would have caused, arriving through a different door. Rebuilding means a fresh WebGL context, which browsers hand out in limited numbers (`phaser4-runtime` P2). The canvas is created detached, the panel adopts it on mount and returns it on unmount, and the game never notices. _[earned 2026-08-11]_

### U15: A live canvas is adopted by the panel that hosts it, not created by it

The renderer's canvas is a plain DOM element made outside React and held by the provider. The panel renders an empty host and, in an effect, appends the canvas on mount and removes it on unmount. Sizing is a `ResizeObserver` on the host, reported up to the provider, which tells the renderer.

**Reason:** this is what makes U9's third case work in practice. The canvas cannot be a React child, because moving it between parents on every tab drag would mean unmounting and remounting the element itself. Two details that are easy to get wrong: the game must be given `parent: null` or it appends its own canvas to the document body (`phaser4-runtime` G5); and everything the panel sends *down* (its size) should be separate from everything it sends *across* (what to draw), because a panel being dragged wider is not a reason to fetch anything again. _[earned 2026-08-11]_

### U18: A second renderer is allowed; the count is bounded by the panel declaration, not by use

The scene viewport and the texture preview are two live Phaser games in one window. That is a deviation from "one game for the life of the window" (`phaser4-runtime` P2) and it is recorded rather than absorbed, because the shape that keeps it honest is narrow: **one game per declared viewport-shaped panel** — booted once, kept for the life of the window, never created per selection, never destroyed when a tab is closed or dragged. The number of live renderers is then a number written in `panels.tsx`, which somebody has to edit on purpose, rather than a number that grows while the human works.

Three practicalities that came with the second one:

1. **Two panels sharing a dockview group means only one of them is rendering.** Putting the Texture tab in the Viewport's group is what makes "the texture opens in its own tab and the scene keeps its place" true, and it also means a hidden panel's body may not be mounted at all. Nothing in a panel may be the only copy of anything (which U9 already required) and no test may assert against a tab that is not in front.
2. **A hidden panel measures 0×0, and reporting that is worse than reporting nothing.** The `ResizeObserver` in the canvas host must ignore a zero box. For a texture that would fit a picture to nothing and then fit it again; for a scene drawn from the bottom edge upward it puts every sprite on a floor line one pixel from the top, and the first real measurement then moves everything — a visible jump for no reason.
3. **Bringing a tab forward belongs to the shell, not to the panel.** A panel behind another tab is not rendering and cannot ask to be looked at, so the effect that says "a texture was selected, show the Texture tab" lives above the layout with a handle on the dockview API. Do both directions or neither: selecting a scene has to bring the Viewport *back*, or half the time the human clicks something and nothing appears to happen. _[earned 2026-08-11, dockview 8.0.0]_

### U16: What a panel draws over a canvas is DOM, and its numbers come from the renderer

Frame guides and the pivot marker are an SVG layer above the canvas, absolutely positioned and `pointer-events: none`. Every rectangle in it is one the renderer reported having cut, at the placement the renderer reported having drawn at; the overlay computes no geometry of its own.

**Reason:** two payoffs and one guard. A one-pixel line stays one pixel at 24× where a renderer-drawn line becomes a 24-pixel bar unless it is drawn in screen space, which means screen-space arithmetic inside a game renderer. Text stays selectable and legible. And it is assertable — a frame count, a marker position and a caption are ordinary locators, which is how a canvas feature gets tested without comparing pixels (`editor-verification` V17). The guard is the reason the numbers come from the renderer rather than from the same inputs: an overlay that re-derives the placement is a second derivation that agrees until it doesn't. _[earned 2026-08-11]_

### U17: Zoom for pixel art is whole steps only, fitting by default

The scale ladder is `1/16 … 1/2, 1, 2, 3, 4, 6, 8 … 32` — always a whole number of screen pixels per image pixel or the reverse. Fitting picks the largest step that fits the panel; the step buttons turn fitting off; a Fit button turns it back on, and so does selecting a different file.

**Reason:** at 3.4× some rows of a sprite are three pixels tall and some are four, which reads as *badly drawn art* rather than as a badly chosen zoom — so the human goes looking for the fault in their own work. Filling the panel exactly is not worth that. Two details: selecting a different file returns to fitting, because a zoom chosen for a 16px sprite is not a choice anybody made about a 4096px tileset; and fitting cannot be decided until the image's size is known, so the first draw lands at whatever scale was current and a follow-up settles it. That settles rather than oscillating for the same reason `editor-kernel` G10 does — after one pass the drawn scale *is* the wanted scale, so the condition is false and nothing runs again. _[earned 2026-08-11]_

### U19: Where the human is looking is per-window and per-scene, held above the layout, and framing waits until the view is worth framing

The scene viewport's camera lives in the provider above the docking layout, as a map from scene path to camera, for the life of the window. Not in the document store, never in a transaction, never serialized (U8) — so Ctrl-Z after a pan reverses the last thing *changed*, by construction rather than by care. Opening a scene for the first time frames it; every later visit finds the view where it was left, including after dragging the tab, which dockview would otherwise reset by unmounting the panel body.

The ladder is U17's, shared unforked with the texture tab so `8×` means one thing across the editor. What is deliberately *not* shared is the mode: a texture preview stays in a fitting mode, and a scene has a one-shot **Frame all** press instead, because a scene camera is driven rather than looked at and a resize has to keep the human's place.

**The interesting part is not where it lives, it is when a scene may be framed.** Framing happens once per scene, so anything wrong at that instant stays wrong until the human touches the view. Three conditions, each of which was a wrong framing before it was a condition:

1. **Every texture has resolved, *and* the report on screen is a report of all of them.** Both halves, and the second is the one that gets missed: a scene's textures resolve one at a time, so the same scene is drawn several times as it opens, and the report being held when the last `.meta` lands is a real report of that scene with a sprite still missing from it. An entity with no picture counts as a point rather than as the area it covers, so a level whose widest object had not arrived is framed on the wrong middle. The fix is to hold *which request* the current report came from and compare by identity — the scene's path cannot answer this, because every one of those reports carries the same path.
2. **The canvas is the size of the panel**, which is a stronger condition than the panel having been measured. A renderer boots at a size of its own and is told the real one when the observer fires, so a scene can be reported drawn on a canvas that has nothing to do with the panel it is in.
3. **The panel has been measured at all** (U18): a tab behind another one is 0×0.

_[earned 2026-08-12]_

### U20: A gesture surface reads pointer and wheel events from the DOM directly, and its keys are the panel's rather than the window's

Middle-drag and space-drag pan; the wheel zooms toward the cursor; `Home` frames everything and `F` frames the selection. That set is what Godot, Unity's 2D view and Tiled have all converged on, plus the space-and-drag habit every art tool since Photoshop has taught. **Left-drag and right-drag are deliberately left unclaimed** — left-drag belongs to placing and selecting, and right-drag is a 3D flythrough idiom that collides with a context menu in a 2D view.

**Reason it is not React's event props:** `onWheel` cannot cancel anything (UG6), and a pan wants pointer capture, which is a DOM call on a real element anyway. The listeners go on the canvas's host in an effect, and the two keys go on the window in a second one, guarded so they never fire while the human is typing (UG7).

One shape worth copying: everything the gesture layer needs arrives as callbacks and it holds no camera of its own, so it is a translator from events to intentions and the state stays in one place (U19). What it *does* own is the two pieces of transient interaction state nothing else can know — whether a drag is in progress and whether space is held — which the panel turns into the grab cursor. Clear the space flag on window blur, or alt-tabbing away mid-gesture leaves the editor believing it is still held and the next left-click pans. _[earned 2026-08-12]_

### U21: One gesture layer decides what a press means, and a drag is travel from the press rather than a sum of steps

Placing an entity by dragging it arrived after panning, into a viewport whose left button had been deliberately left unclaimed. The tempting shape is a second pointer layer on the same element for the new button; it is wrong, and the first thing it breaks is space-drag starting on top of a sprite — two listeners racing to interpret one press, with the loser's gesture simply not happening. So the rules stay in one hook, in priority order: space wins, then whatever is under the pointer, then empty space.

Three things that are not obvious until the second gesture exists:

1. **Selection happens on the press, not the release.** Not for feel — because the press is also the only moment the panel can record *where the entity was*, and it needs that for the next point.
2. **A drag is applied as travel from the press**, never as a running sum of the increments between moves. With snapping on, adding up rounded steps lets a sprite creep away from the pointer over a long drag and never come back. The same shape as any accumulate-versus-recompute choice, and the symptom here is a sprite that no longer sits under the cursor.
3. **A modifier is read off each move event**, not remembered from the press, so it can be taken or let go mid-gesture. It also needs somewhere to be *said*: a caption that appears only during the drag is the only place a human will ever find out the modifier exists.

The threshold matters too — a press that never travels more than a few pixels is a click and must not nudge what it selected. _[earned 2026-08-12]_

### U22: A control that makes a file shows the whole path before it commits, and takes the destination from the selection

The Assets panel gained a name field, a button, and a line reading "Will make `scenes/level-03.json`" that updates as the name is typed. Three rules came out of building it:

1. **The destination is the selected folder** — or the selected file's folder, or the top of the project — and is looked up in the tree rather than guessed from the path string. **No folder name is written into the editor.** `scenes/` is a convention in the folder map and not a fact this code is allowed to rely on, the same ordering as U11: the editor decides what a file is from what it says, and where a file goes from what the human picked.
2. **The whole path is on screen before anything is committed.** This is the one control that puts a file in somebody's project, and "where did it go?" should never be a question answered by searching. It is also the only affordance that explains the destination rule without a paragraph of help text.
3. **The refusal is the service's own sentence, shown as it arrives.** The service knows things the panel does not — whether the name is taken, whether the folder exists — and paraphrasing them in the panel is two descriptions of one rule, of which one will go stale.

The consequence to say out loud rather than paper over: making a file is not an edit to a document, so **Ctrl-Z does not take it back** (`editor-kernel` D7). _[earned 2026-08-12]_

### U10: A panel always says something, and "nothing to show" is a sentence rather than a blank

Every state gets prose: a folder, a document whose format has no inspector yet, a file the editor does not import, an asset whose settings have not landed, a settings file that will not parse, and nothing selected at all. **Reason:** a blank panel is indistinguishable from a broken one, and in a kernel where most formats do not exist yet, "there is nothing here" is the *common* case rather than the exceptional one. Two things that make the sentences worth reading: name what the thing is (a scene, a sound) rather than what it lacks, and say what would change it ("its own inspector arrives with the scene format"). When settings cannot be parsed, **show the file's text** — being told a file is unreadable without being shown it forces the human out of the editor to find out why. _[earned 2026-08-11]_

**Generalised from the inspector to any panel, by the viewport needing the identical rule.** Two things the second application made explicit. **Nothing-selected and this-is-not-the-right-kind-of-thing must be different sentences**, because being able to tell them apart is the entire value — "Select a texture to see it here" and "hit.wav is a sound; the viewport draws textures" are different situations and collapsing them into one blank loses both. And **the previous answer is cleared when the new one is a sentence**: a viewport that leaves the last picture up while the caption talks about a sound is worse than an empty one, because it looks like it is working. _[earned 2026-08-11]_

### U11: What a file says it is beats what its name suggests

The inspector shows the type from the `.meta` when there is one and falls back to the extension only when there is not. **Reason:** the sidecar is authored and the extension is a guess, so preferring the guess would make a deliberate override invisible. The same ordering applies anywhere else a document and a filename both claim to describe the same thing. _[earned 2026-08-11]_

### U12: A control is built over the document in the store, never over the answer that was served

The Inspector shows a texture's settings from the document store. The fetched `MetaView` is still what decides the states that have nothing to edit — no settings file, one that will not parse — but where the document itself is absent the panel says it is reading rather than rendering a control.

**Reason:** the transaction API can only change a document the store is holding, so a control built over the served answer looks entirely correct and does nothing when used. That is the worst failure available in a panel: no error, no exception, and nothing visually different from a working control. It reads as "the setting doesn't stick", which sends the next session looking at the file writer, which is fine. Rendering the control only when the document exists makes the failure unreachable rather than merely unlikely — the same instinct as `editor-kernel` G2, applied one layer out. _[earned 2026-08-11]_

### U13: Undo is a window-level shortcut, and it takes Ctrl-Z away from text fields

`keydown` on the window; Ctrl/Cmd-Z undoes, Ctrl-Y and Shift-Ctrl-Z redo, `preventDefault` unconditionally — including with the cursor in a text input.

**Reason:** there is one undo stack for the whole project (`editor-kernel` D7), so the key has to mean the same thing wherever the cursor happens to be. Leaving the browser's own field-level undo in place would make Ctrl-Z step back through the individual keystrokes of a value the document only ever saw as one change, so the same key would do two different things depending on where you were pointing. Every editor of this kind makes the same trade. Worth stating plainly because it is not a nicety: "one press of Ctrl-Z, not one per digit" is only satisfiable this way. _[earned 2026-08-11]_

### U14: A number field holds its own text while it is being edited, and yields to any value that arrives from elsewhere

It renders what has been typed while that text is in play, and the document's value otherwise. Text that is not a number yet is held and not committed; anything that is, is committed on the keystroke. Leaving the field drops any leftover text and seals the undo step.

**Reason:** a number input fed straight from the document cannot be emptied to type a new value — clearing it parses as nothing, the document keeps the old number, and it reappears under the cursor mid-edit. But the field must still lose to the document when a value arrives from somewhere else, or undo and a text-editor save both appear to do nothing while the field has focus. Committing per keystroke rather than on blur is what keeps "on disk within a second" true during a long edit; the undo step stays single because of the merge key, not because the commit was held back. _[earned 2026-08-11]_

### U23: A panel that shows a *resolved* document reads one thing and writes another, and says which is which

With prefabs, the level on screen is not the level in the file: an instance's picture comes from somewhere else. Every panel that draws or describes a scene now reads the resolved entities, and every panel that changes one still edits the document — the Viewport, the Hierarchy and the Inspector all hold both, and each names them apart (`entity` versus `resolved`).

**Reason:** the resolved copy carries components the file does not, so writing one back bakes them in and severs the link to the prefab, silently. What makes that unreachable rather than merely avoided is a habit that was already universal for another reason: **every recipe re-finds its target by id inside the transaction** (`editor-kernel` D7) and therefore can only ever touch the document. Two practicalities that only show up once you build it. Pass both into a component rather than resolving inside it, so "which one is this?" is answerable at the call site. And **fall back to the file's own entity while resolution is in flight** — the alternative is a row that vanishes for a render, which reads as a bug in the list. _[earned 2026-08-12]_

### U24: A gesture that selects what it just made must be reachable from where it lands

Placing a prefab selects the instance it placed — which moves the Inspector off the prefab, so the button that placed it is gone and "place it fifty times" is fifty round trips through the Assets panel. The fix is not to stop selecting: it is that the thing you land on offers the same gesture. The placed instance's panel carries a **Place another** beside the name of the prefab it came from.

**Reason:** the Inspector holds one thing at a time, and that is the right design — but it means any action that both *creates* and *selects* moves its own control out of reach. This is a shape, not a one-off: duplicate, place, add-from-template and paste will all hit it. The test is worth stating as an acceptance criterion in its own right, because pressing the button once always works and nothing in a single-press test can see the problem: **press it twice without touching anything else.**

Selecting the result is still right — the outline lands on the new thing and one drag moves it — so the answer is a second door, not a removed one. And the two doors must be one implementation: a hook, taking what it operates on as an argument. Two copies of "make an instance" is two chances to write one that copies what it should reference. _[earned 2026-08-12]_

### U25: A control that makes a file names the file; what goes inside it is the button

Making a level and making a prefab share one name field, one folder rule and one path preview, and differ only in which button is pressed. Nothing about the name or the folder says which kind it is.

**Reason:** the *path* is the same question either way, and asking it twice in two rows would imply the answers differ — that prefabs go somewhere levels do not. They do not: `prefabs/` is a convention in the folder map and the editor is not allowed to rely on it (U11's rule, one level up from a filename to a folder name). Keeping the choice in the button also keeps the promise honest, since the path preview stays true whichever is pressed. The name typed becomes the name *inside* the document as well, which is the only sensible default and is editable the moment it opens. _[earned 2026-08-12]_

## Gotchas

### UG6: React's `onWheel` cannot `preventDefault`, and a middle-button `mousedown` must

Two ways a viewport gesture fails silently, both fixed in the same effect.

**The wheel.** React registers its root-level wheel handling as *passive*, so `event.preventDefault()` inside an `onWheel` prop does nothing whatever — no error, no warning in production. The visible symptom is not the wheel: it is that a trackpad pinch (which arrives as a wheel event with `ctrlKey`) zooms the entire editor UI. **Fix/policy:** attach it by hand — `element.addEventListener('wheel', handler, { passive: false })` in an effect — and cancel unconditionally, including the ctrl-wheel. Worth checking on sight anywhere a component wants a wheel to mean something other than scrolling.

**The middle button.** Chrome on Windows opens its autoscroll widget on a middle `mousedown`, and preventing the corresponding `pointerdown` does not reliably stop it. **Fix/policy:** a separate `mousedown` listener that cancels `button === 1`. The symptom without it is a pan that turns into a page scroll with a compass glyph sitting over the level. _[earned 2026-08-12, React 19.2.8]_

### UG7: A bare-letter shortcut fires while the human is typing, because the existing handler never had to care

The editor's only keyboard handler was Ctrl/Cmd-only (U13), so it could take the key wherever the cursor was and that was the point. The moment a viewport wants `F` for "frame the selection", the same reasoning inverts: an `f` typed into an entity's name must be an `f`. **Fix/policy:** every bare-key handler checks the event target first — `INPUT`, `TEXTAREA`, `SELECT` or `isContentEditable` means the key is not yours. `Space` needs cancelling as well as ignoring when it *is* yours, or it scrolls the page and re-presses whichever button last had focus. _[earned 2026-08-12]_

### UG9: A surface that takes presses must take focus, or its keys land in the field the human was last typing in

The viewport cancels the default on `pointerdown` — to stop text-selection drags — and cancelling it also stops the browser moving focus. So after typing in the Inspector, clicking a sprite leaves focus in that text field, and the next press of `F` is an `f` in somebody's entity name. Nothing errors; the level simply gets renamed a letter at a time.

**Fix/policy:** give the surface `tabIndex={-1}` — focusable, never in the tab order — and call `element.focus({ preventScroll: true })` on the press, with `:focus { outline: none }` so it does not sprout a ring. Then UG7's typing guard answers correctly, because the human genuinely has stopped typing. Worth checking on sight in any panel that both cancels `pointerdown` and owns a bare-key shortcut: the two are individually reasonable and wrong together. _[earned 2026-08-12]_

### UG8: A caption under a canvas changes the size of that canvas, which is a feedback loop as soon as anything is computed from it

The scene viewport is a stage above a caption bar in a flex column, so a caption that wraps to two lines makes the canvas twenty pixels shorter. Harmless for years — until framing, which is computed against the canvas size. The sentence that says "everything is off screen" is on screen at *exactly* the moment the human presses the key that frames the level, so the level gets framed for a panel that is about to grow back the instant the sentence goes away. Two presses of one key, two different zooms.

**Fix/policy:** the caption on a surface whose size is used for anything must not be able to change that size. Switch wrapping off and let the notes ellipsise, with the full text on `title`; keep the sentences short enough that nothing is actually clipped. Shortening the sentence alone is not the fix — it moves the threshold to a different panel width. Worth checking on sight whenever a status line sits inside the same box as something measured: the tell is a message that only appears in the state the measurement is about to change. _[earned 2026-08-12]_

### UG1: Dockview's layout is computed inside `requestAnimationFrame`, so a non-compositing surface freezes it at 100×100

Dockview sizes itself from a `ResizeObserver` whose callback is deferred through rAF. In any browser surface that is not painting — a hidden preview pane, a background tab, a devtools-detached window — rAF never fires, so the grid keeps its initial 100×100 size, every panel is 100px wide, and every sash reports `dv-disabled`. It looks exactly like a broken layout or a missing CSS import. **Fix/policy:** before debugging dockview sizing, confirm the surface is actually compositing; verify in a real window or in headless Chromium (which does paint). Nothing about the CSS or the container needs changing. _[earned 2026-08-11, dockview 8.0.0]_

### UG2: dockview-react 8 logs a console error about `dockview-enterprise` on every load

`DockviewReact` unconditionally passes a `createContextMenuItemComponent` framework option, and dockview-core logs an error when the matching enterprise module is not registered. It appears once per page load, is not caused by anything the consumer does, and cannot be switched off through props. **Fix/policy:** ignore it, do not install `dockview-enterprise` to silence it, and do not write a "no console errors" assertion into the browser suite — that assertion would be permanently red for an upstream cosmetic. _[earned 2026-08-11, dockview-react 8.0.0]_

### UG3: Vite binds to `localhost`, which is IPv6-first on Windows

The dev server's default host resolves to `::1`, so anything looking for the editor at `127.0.0.1` — a test harness, a health check, a script — times out against a server that is demonstrably running and printing its URL. **Fix/policy:** set `server.host` to `127.0.0.1` explicitly, the same literal the sidecar binds to. Spell it as an address, never as `localhost`, anywhere the two halves have to find each other. _[earned 2026-08-11, Vite 8.2.1]_

### UG5: A hook whose state is cleared by an effect answers for the *previous* selection for one render

`useAssetMeta` cleared its state in the effect that re-fetches, and effects run after the render that changed the selection — so for exactly one render it answered with the file selected a moment ago. One render is long enough to matter twice over: the panel shows one file's settings under another file's name, and anything clicked in that render belongs to neither. It is also the render a browser test can land in, so the bug hides behind a green suite (see `editor-verification` W7).

**Fix/policy:** hold the answer *and* the thing it is an answer about, and compare them at render time — `answer.forPath !== path` means loading, whatever the effect has or has not done yet. Never rely on an effect to invalidate state that a render can already tell is stale. Worth checking on sight in any hook of the shape "fetch when this prop changes": if the only thing clearing the old value is inside `useEffect`, the stale render exists. _[earned 2026-08-11, React 19.2.8]_

### UG4: Panels have close buttons and the kernel has nowhere to reopen them from

Dockview tabs render a close affordance by default, and the shell has no panel menu and no layout persistence, so a closed panel is gone until the page is reloaded. Reloading restores the default layout, which makes this shallow rather than dangerous — but it is a real dead end for a human who does not know that. **Fix/policy:** it is fixed by the feature that adds a panel menu plus saved layouts, not by hiding the button; hiding a control while its keyboard and context-menu paths still work is worse than leaving it visible. Until then, say so in the hand-off. _[earned 2026-08-11]_

## Contracts

Contracts are referenced as file paths, never paraphrased as prose. Read the file; don't trust a summary of it.

- `kernel-2d/editor/shell/panels.tsx` — every panel the editor has and the layout it opens in. Adding a panel happens here and nowhere else (U1), and the count of live renderers is bounded by what is in here (U18). A panel gains a real body by getting a `render`; without one it shows its own description, which is what keeps unbuilt panels honest instead of blank. All five have bodies now.
- `kernel-2d/editor/panels/AssetsPanel.tsx` — the folder mirror, the worked example of a panel with a body, and U22: the one control that puts a file in a human's project.
- `kernel-2d/editor/panels/InspectorPanel.tsx` — the worked example of U10, U11 and U12: every state it can be in has a sentence, and the editable one is reached only from the store.
- `kernel-2d/editor/panels/TextureSettings.tsx` — the first editable controls, hand-written, and the worked example of U14. Every one of them goes through the transaction API and none of them knows undo exists.
- `kernel-2d/editor/shell/useUndoShortcuts.ts` — U13, and the only keyboard handler in the editor.
- `kernel-2d/editor/panels/ViewportPanel.tsx` — the scene, and the worked example of U10 at its widest: no scene open, opening, gone, unreadable, empty, a texture it cannot draw, everything off screen, and the selected entity off screen are eight different sentences. The last two arrived with the camera, and they *replace* the count rather than joining it — see UG8 for why that is layout as well as prose.
- `kernel-2d/editor/panels/TexturePanel.tsx` — U15 and the single-texture preview, moved out of the Viewport when the Viewport became the scene.
- `kernel-2d/editor/panels/TextureOverlay.tsx` — U16. Frame guides, the strip no frame reaches, the pivot marker, and the caption that says in words what the shading says in pixels.
- `kernel-2d/editor/panels/SceneOverlay.tsx` — U16 again: the selected entity's outline, the crosshair on its position, wherever the camera has put the corner the scene's y counts up from, and which entities are actually on the canvas.
- `kernel-2d/editor/panels/HierarchyPanel.tsx` — what is in the open scene, in draw order, and the four actions that change it — every one a transaction and none of them aware undo exists.
- `kernel-2d/editor/panels/EntityInspector.tsx` — the second inspector, and where a D5 reference is written.
- `kernel-2d/editor/panels/NumberField.tsx` — U14, shared the moment a second inspector wanted the same behaviour.
- `kernel-2d/editor/shell/viewport-context.tsx` — U9's third case: the texture renderer, above the layout, and the zoom state of U17.
- `kernel-2d/editor/shell/scene-view-context.tsx` — U18 and U19: the scene renderer, why two is a bounded number rather than a habit, one camera per scene for the life of the window, and the three conditions a scene satisfies before it is framed.
- `kernel-2d/editor/shell/useSceneGestures.ts` — U20 and U21: left-press to pick and place, middle-drag, space-drag, wheel-to-zoom, the two framing keys, and the order they take priority in.
- `kernel-2d/editor/shell/drawn-entities.ts` — every question asked about the picture the renderer drew: what an entity covers, what is on the canvas, and what is under the pointer. One set of rectangles, so a click cannot disagree with an outline.
- `kernel-2d/editor/shell/open-scene.tsx` — which scene is open and the document behind it; one fetch that both decides a `.json` is a scene and puts it in the store.
- `kernel-2d/editor/shell/scene-assets.tsx` — U9 for a set whose membership changes: every texture a scene refers to, resolved once per window.
- `kernel-2d/editor/shell/layout-context.tsx` — the handle on the docking layout, and the only thing that reaches it from outside a panel.
- `kernel-2d/editor/shell/useSelectionFocus.ts` — U18's third practicality: which tab comes forward for what was just clicked.
- `kernel-2d/editor/shell/asset-meta-context.tsx` — U9's second case, and why a fetch with a side effect is never a panel's to own.
- `kernel-2d/editor/shell/zoom.ts` — the scale ladder of U17.
- `kernel-2d/editor/shell/asset-kinds.ts` — what a file is (U11) and how to find it in the tree. Shared the moment a second panel needed the same answer.
- `kernel-2d/editor/shell/asset-rows.ts` — which rows a folder has, and why a sidecar folds into the row of the file it annotates (`editor-kernel` D4). Shared because the Inspector counts a folder the same way the panel lists it.
- `kernel-2d/editor/shell/selection.tsx` — U8. What is selected, which scene is open, and nothing else.
- `kernel-2d/editor/shell/scene-prefabs.tsx` — U23's source: the resolved level every panel draws and describes, and the loud note saying it is never the thing that gets written.
- `kernel-2d/editor/shell/useReferences.ts` — following a reference by path and modification time, once: the read-token ordering, the ask-once rule, and why "not ready" is not a problem. Shared by the textures a level draws and the prefabs it places from.
- `kernel-2d/editor/panels/fields.tsx` — the four pieces every inspector body is built from, and the note on which lookalike deliberately did not move in.
- `kernel-2d/editor/shell/entity-names.ts` — one free-name helper, and the two things that actually differ between its callers.
- `kernel-2d/editor/shell/usePlacePrefab.ts` — U24 as built: one gesture, reachable from the prefab and from what it just placed.
- `kernel-2d/editor/panels/TexturePicker.tsx` — one picker, two owners, and the D5 round trip that fetches a texture's id. Lifted out the moment a prefab needed the same control as an entity.
- `kernel-2d/editor/shell/project-context.tsx` — U9. One folder read and one change stream per window.
- `kernel-2d/editor/shell/useAssetMeta.ts` — one file's import settings, re-asked when the folder changes as well as when the selection does.
- `kernel-2d/editor/shell/useProjectTree.ts` — the folder, kept current: the re-read of U5, the settle of U6, and the stale state of U7.
- `kernel-2d/editor/shell/App.tsx` — the shell: status strip above, docking layout below, providers around both.
- `kernel-2d/editor/shell/useSidecarStatus.ts` — how the editor learns which project it is connected to, and what it does when the answer stops coming (U3).
- `kernel-2d/editor/shell/StatusStrip.tsx` — the connection line, including the `data-testid` hooks the browser suite reads.
- `kernel-2d/editor/shell/shell.css` — the frame around the docking layout. Dockview's own chrome comes from its theme, not from here.
- `kernel-2d/vite.config.ts` — the editor's root folder, the `/api` proxy to the sidecar (U2), and the loopback binding (UG3).
- `kernel-2d/scripts/editor-server.ts` — the editor window's host, port, and open-a-browser knobs, with their environment variable names.
- `kernel-2d/tsconfig.editor.json` and `kernel-2d/tsconfig.base.json` — the browser half of U4.

- `kernel-2d/editor/store/open-documents.ts` — the one document store per window and the hooks panels read it through. The API itself is `editor-kernel`'s (D7); this is where the UI meets it.

**Not yet written** — until a path appears here, the contract does not exist and must not be assumed:

- Inspector auto-generation from Zod schemas. There are two hand-written inspectors now, which is the point at which generalising has something to generalise *from*.
- Gizmos for rotate and scale, a marquee, dragging a pivot or a frame grid, and cycling through overlapping sprites with repeated clicks. Position is the one thing a viewport can change; everything else is the Inspector's, deliberately.
- A grid, rulers, or a settable snap size. A drag lands on whole level units and Alt frees it; nothing is configurable and nothing is drawn.
- Dragging inside the Hierarchy tree, and nesting. The list is flat and reordering is two buttons.
- Multi-selection. `Selected` is a union of one thing; making it a list is a change to that type and to every reader of it.
- A panel menu or anything that lists the viewport's shortcuts. `Home` and `F` are named in the caption's own sentences and on two buttons, and nowhere else.
- Two scenes open at once. `openScene` is one path.
- The panel menu and saved layouts (UG4).
- A keyboard shortcut registry. There is exactly one handler (U13), and one does not need a registry.
- Anything that shows the undo stack — a history panel, an Edit menu, an "Undo <label>" caption. The labels exist and `peekUndo`/`peekRedo` expose them; nothing reads them yet.
