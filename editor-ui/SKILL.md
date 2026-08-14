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

### U26: Play mode is window state, and a mode that suspends editing suspends it in two places at once

Which level is running, the picture it was started from and whether it matched all live in a context beside the selection and the camera (U8) — never in the document, never in a `.meta`, never in `project.json`. Ctrl-Z cannot see it, and stopping leaves no trace that it happened.

**While it runs, every panel but the Viewport carries `inert`, *and* the module that exports the transaction API refuses an edit.** Two mechanisms, deliberately:

- **`inert` is what the human sees**, and it beats a `disabled` on each control: it covers the whole subtree including whatever a later session adds, and it takes the panel out of the keyboard order, which a pile of `disabled` attributes would each have to remember. The wrapper is `display: contents`, so no box is added and the docking layout is untouched.
- **The refusal is what makes the guarantee true** whichever control anybody finds a way to reach. It gates the editor's own module (`open-documents.ts`), not the transaction API itself: play mode is a fact about this window, not about the document model, and `documents.ts` should stay readable by somebody who does not know play mode exists. **Undo is gated with the edits** — Ctrl-Z is a change to a document like any other and is written to disk like any other. `adoptFromDisk` stays open, because a file changing underneath the editor is a read.

Two details that only show up on screen. **`inert` is invisible**, so the dimming is not decoration: without it the Hierarchy looks exactly as it does when it works, and the answer to "why is Delete doing nothing" is a caption under a canvas somewhere else. Style it off the attribute, never off a class of your own, so what is out of reach and what looks out of reach are the same set by construction. And **the explanation belongs in one place** — the panel that took the mode over — rather than repeated as a note in each of the four panels that went quiet.

**Reason:** a mode that changes what the whole window will accept has to be visible from wherever the human's hand is and provable from wherever the code is. Neither half does both. _[earned 2026-08-12]_

### U27: "There is a picture" and "the picture is of what I am asking for now" are different questions, and features keep needing the second

A level's textures resolve one at a time, so the same level is asked for several times as it opens and the report in hand is regularly a real report of that level with half of it missing. There is also a render between the last texture arriving and the redraw being *requested*, during which everything looks ready and the picture on screen is of the request before last.

So the renderer's context answers the sharper question directly: `useDrawScene` hands in a key with each request and answers whether what is on screen came from *that* key. Framing waits for it; so does the Play button.

**Reason:** two features reached for the loose version first and were wrong the same way a session apart. Framing a level against half of itself puts it off-centre permanently, and was patched with three ad-hoc conditions inside an effect. Play offered a render too early starts against a baseline the human never saw — so the comparison that is the entire point of the feature checks the running level against a half-drawn one. The tell is a test that passes alone and fails in a full suite run: the race widens when the machine is busy, which is also when nobody is watching. Answer it once, in the context that actually knows, instead of deriving it from "is there a report" at each call site. _[earned 2026-08-12]_

### U28: A third inspector body cost nothing, and the pattern that made it free is worth naming

Project settings — which level the game starts on — became an editable body in the Inspector rather than a panel of its own. The whole change was one branch in `DocumentBody`, keyed on the `format` the document carries, plus a component beside the prefab's and the entity's.

Three things it confirms, each of which was a decision made earlier and is now evidence rather than intention:

- **No new panel.** The reachable route already existed: click the file, the Inspector describes what is in it. A *Project* panel would have added a permanent box to the default layout for a setting somebody touches once a month, and U1's declaration file is where the cost of a panel is paid.
- **Keyed on what the document says it is, never on its name.** `project.json` is a convention in the folder map; the branch reads the `format` literal (U11 again, and `editor-kernel` D22's registry). A human who keeps their settings somewhere else still gets the controls.
- **Undo, autosave and disk-wins arrived for free**, because it is an ordinary document going through the transaction API. Worth stating in the hand-off as a *capability* — "Ctrl-Z takes the choice back" — since the human has no way to know that fell out of the architecture rather than being built.

**And a reference with no id to fetch is validated by reading the file, not by trusting the path.** `TexturePicker` is asynchronous because it fetches the texture's id (D5). This picker is asynchronous for a different reason: there is no id to fetch, and nothing about a `.json` path says whether the file at it is a level. So the list offers every document in the project and the *pick* reads the file — refusing "enemy-slime.json is a prefab, not a level" instead of writing a path the export would have to complain about later. The list stays wide on purpose: narrowing it would mean reading every `.json` in the project up front, to save one read of the one that was chosen.

The picker is **not** lifted into a file of its own. There is exactly one thing in the kernel that points at a level; U25's sharing rule ran the other way here, and a shared component built for one owner is a guess about what two would have in common. _[earned 2026-08-12]_

### U29: Renaming and moving are one control, and a destructive control's first press is a sentence

The Assets panel gained a second row under U22's: the selected file's name, the folder it is in, the whole destination path, one button, and a Delete beside it.

**Renaming and moving are one operation with two ways to ask for it.** To the filesystem and to the reference fixup there is no difference — a file that was at one path is now at another — so two rows would imply the answers differ, and the human would have to learn a distinction the editor does not have. One name field, one folder, one path preview, and the *word on the button* changes to say which of the two this will be (`Rename` when only the name moved, `Move` otherwise). Same instinct as U25 and pointed the other way: there, one path question with two buttons because the *contents* differed; here, one path question and one button because nothing does.

**U22's path preview matters more for a file that already exists than for one being made.** Making a file in the wrong place leaves you with a file in the wrong place; moving one leaves you looking for something that used to be findable.

**Delete takes two presses, and the first one is the sentence rather than a confirmation.** It runs the reference walk and says what still points at the file — "knight.png is still used once, in scenes/level-01.json" — and the button becomes *Delete anyway*. Three options were live and the other two are both worse:

- **Refusing while something uses it** breaks the commonest reason to delete an asset, which is that a better version is going in its place, and sends the human back to the file manager — where nothing gets fixed up at all. A tool that refuses the normal case teaches people to work around it.
- **Deleting silently** is defensible only because the level already says what is missing; it throws away the one moment the answer is cheap and the human is still deciding.

A second press is also what stops Delete being one stray click, so the sentence is shown even when nothing uses the file. **When a destructive control needs a confirmation anyway, spend it on telling them something they did not know.**

Two smaller things worth copying. **Key the whole row on the selected path** rather than clearing four pieces of state when the selection changes — a remount cannot leave somebody else's typed name, refusal or half-pressed Delete under a different file, and it is the same class of bug as UG5 answered structurally. And **do not offer a destination that will be refused**: the folder list leaves out the folder being moved and everything under it, so "a folder cannot be moved inside itself" is a service rule the human never has to be told. _[earned 2026-08-12]_

### U30: A file operation moves what the human is *looking* at, and none of it is undoable

Renaming the open level, or the selected file, has to carry the selection, the open scene and that scene's camera to the new path. All three live outside the document store (U8, U19), so none of it is in a transaction and none of it is reversed by Ctrl-Z — which is exactly right, and is why it has to be done deliberately rather than falling out.

The camera is the one that would have been missed, and its symptom is the confusing one: cameras are keyed by scene path, so a renamed level arrives somewhere nothing has ever been framed and gets framed again. **The level jumps, for a reason that has nothing to do with looking at it**, and "each level remembers where you were looking" quietly stops being true for exactly the levels somebody has been reorganising. A per-scene map keyed on a path needs a rename hook the moment paths can change; it is five lines and there is no test that would have caught its absence.

The exception that proves the rule: **the tab that comes forward is not remapped, because it was never keyed on the path.** Selecting a texture brings the Texture tab forward (U18), and a renamed texture is still a texture, so it does the right thing by itself. Ask of each piece of window state: is it keyed on the path, or on what the thing *is*? Only the first kind needs moving. _[earned 2026-08-12]_

### U31: A snap needs two numbers, and a repeat-place mode is keyed on a path rather than on the selection

Two features, one entry, because they were built together and neither is worth much alone: a settable snap makes twenty-six placements line up, and placing by clicking is what makes twenty-six of them bearable. Both are settings of the *surface*, so both live above the layout beside the camera and the selection (U8, U19, U9) — never in a document, never in a `.meta`, never in `project.json`, invisible to Ctrl-Z, and back to their defaults on reload.

**The snap is a step *and* an offset, and leaving the offset out is the mistake to have already made.** A step alone describes a grid through the origin, reaching 0, 16, 32. The first real board asked for 8, 24, 40 — because a sprite hangs off the *middle* of its position, so a 16-unit tile covering the square from 0 to 16 sits at 8. A step-only snap does not merely fail to help there: it puts every new tile half a cell from the ones already drawn, which is near enough to look deliberate and far enough to be a different square. Godot's 2D editor settled on the same pair (grid step, grid offset) and this is why. The default is `1 from 0`, which is exactly how the editor placed before any of it existed, so a window nobody has touched behaves as it always did.

**A repeat-place mode holds the path of what it is placing, not the selection** — and that is the whole design, not a detail of it. U24 established that placing selects what it placed, which is right for one placement and impossible for twenty: the Inspector holds one thing at a time, so the first click would move it off the prefab. So the mode remembers a path, and **a press while it is on selects nothing at all.** That is a deliberate exception to U24 rather than a contradiction of it — U24's problem was a control moving out of reach, and the answer there was a second door; here the answer is for the gesture not to move anything. The test is the same shape and worth writing as an acceptance in its own right: **click three times without touching anything else.**

Four things that only show up once it is built:

1. **The mode has to beat "whatever is under the pointer" in the press order**, not lose to it. A board being drawn is covered by its own backdrop, so nearly every press that means "another one here" lands on something. A mode that placed only into empty space would place nothing at all on the first level anybody tried. Space-drag still wins over both, so panning works mid-board.
2. **A mode that outlives the panel that switched it on has to say so somewhere that follows the eye.** The switch is in the Inspector and the hand is in the viewport, so the viewport's caption names what is being placed and the key that stops it, and the cursor changes. Esc is the way out, guarded by UG7's typing check like any bare key.
3. **It is state keyed on a path, so U30 applies again, mechanically.** Rename the prefab you are half way through drawing a board with and every further click silently places nothing; delete it and the same. Both are handled where the camera's remap already is, and both are five lines that no test would have missed by their absence.
4. **Turning the mode on must not place one.** The button beside it already means "one, now, in the middle of the view"; a toggle that also placed one would put the first copy somewhere the human did not click, which is the thing they turned it on to stop.

**What this did not become, and the reason is the interesting part.** Drawing a road was expected to need a tile painter — a panel, a grid, a brush. It turned out to need a snap and a press, both genre-neutral, both about a hundred lines, and the tile painter never had to exist. The rule that produced that (`genre-spinup` S1) sends work at the game's own vocabulary rather than at a panel; the corollary this session adds is that **the kernel's own gestures getting better is what tedium is answered with**, and a genre tool is for what is impossible rather than for what is slow. Drag-to-paint was not built for the same reason: clicking is quick enough, and a stroke that stamps is the first step toward a brush nobody has asked for. _[earned 2026-08-13]_

### U32: One accent, it means "current", and it is defined in one file — dockview's included

The editor was repainted to a reference UI the human supplied: near-black grounds, a hairline between surfaces, one warm accent. What survives the specific palette is the structure underneath it.

**Every colour in the editor is a token in `kernel-2d/editor/shell/shell.css`, and the docking library's are the same tokens.** Dockview themes are built from a handful of base colours the rest derive from — Abyss has five — so a repaint is redefining those five under `.dockview-theme-abyss` in this stylesheet. The alternative, authoring a bespoke theme object in TypeScript, splits the palette across a `.css` and a `.tsx` and guarantees the two drift. The theme is still chosen in `App.tsx`, but for its *structure* rather than for its colour.

**The accent means one thing: this is the current one.** The open tab of the focused group, the selected row, the field being typed in, the zoom that is exactly fitting. Not buttons, not headings, not anything merely important. It is the only saturated thing on a near-black screen, so a second meaning costs it the first — and "wrong", "waiting" and "working" keep their own three colours, which is what lets a status dot and a selection sit on screen together without being read as related.

Three things that only appear once it is on screen:

1. **"The current tab" has to mean the focused group's, not every group's.** Four docked groups have four open tabs; outlining all four says "current" four times, which says it none. The qualifier is one selector (`.dv-active-group`), and without it the rule above reads as decoration.
2. **Selection is stated twice — a wash across the row and a bar down its leading edge.** On a near-black ground a 14%-opacity wash sits within a hair of an ordinary hover, and an edge bar alone is at the margin where nobody is looking. Either alone reads as "did that click do anything?"; together they do not.
3. **Type carries a role split: sans for words, mono for chrome.** Tabs, status strip, section headings, badges, and the fields values are typed into are mono; file names, sentences and captions are sans. It is worth doing because it makes chrome unmistakably chrome — and it is worth doing carefully, for the reason in UG10.

**A panel reads as a window with tabs on it, and that shape is three colours in the right order.** `background` is the frame the panels sit in and shows through the gaps between them; `strip` is the bar a panel's tabs sit on; `surface` is the page inside it — and **the open tab is painted `surface`**, square along its bottom edge with no margin under it, so it *is* the page reaching up into the bar. That is the whole of the browser-tab effect; it survives any palette that keeps those three in that order and dies the moment the open tab is given a fill of its own. Two consequences that are not obvious:

- **The accent goes on the tab's top edge, not around it.** An outline draws a line along the bottom too — between the tab and the page it is supposed to be part of — and undoes the join.
- **A panel whose content is not the page colour breaks the illusion**, and the viewport is exactly that: a near-black canvas well. Fixing it by lightening the canvas would be the tail wagging the dog, so the well is inset a fixed eight pixels instead, and the page shows as a frame around it. Fixed, because UG8 — the canvas's size is what framing is computed against, so an inset that varied with the panel's contents would be a feedback loop.

The structure comes from picking dockview's **spaced** theme (`themeAbyssSpaced`) rather than hand-rolling gaps and radii: it is the variant that already gives each group rounded corners, a gap, and tabs belonging to their own group. Choose the theme for its structure and repaint its five colours; do not fight its structure with CSS.

_[earned 2026-08-13]_

## Gotchas

### UG10: A monospace font on every control clips the buttons, and only a screenshot says so

Putting the chrome's mono face on `.control` — one line, since every control shares that class — clipped "Duplicate" to "Duplicat" and wrapped a three-button row onto three lines each. Mono is roughly a sixth wider than the UI sans at the same size, and a button is sized by its label where a field is sized by its box.

**Fix:** mono on the fields (`--number`, `--text`), never on `--action`, `--step` or `--choice`.

**Why it is worth an entry:** the full suite stayed green through it. Nothing asserts a button's width, and nothing should — that is a test that fails on every font change and catches nothing else. What caught it was looking at a screenshot of the panel, which is what the definition of done means by screenshot-verified rather than test-verified. **A type change is a layout change wherever a box is sized by its text.** _[earned 2026-08-13]_

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

**Wrapping was only the first way to break it, and the real rule is stronger: the bar must be the same height whatever is in it.** Putting two number fields in that bar made the canvas shorter *while editing* than while playing — a text input's baseline sits lower in its box than a paragraph's, so under `align-items: baseline` adding one grew the line box, and play mode swaps the whole caption for a different one. The canvas therefore resized at the instant Play was pressed, and the check that the running level was drawn exactly as the editing view had it refused to compare two different canvases (V24, correctly) and reported **"cannot be checked"** — which reads as a limitation of the feature and is a layout bug three files away.

**Fix/policy:** centre rather than baseline-align, and give the bar a floor at the height of its tallest control, so two captions holding different things still measure the same. The general form, and it is why this is worth an amendment rather than a line: **when two modes swap one box for another, the box's size is part of the contract between them** — and the thing that notices is never the layout, it is whatever downstream feature compares the two modes. _[earned 2026-08-13]_

### UG1: Dockview's layout is computed inside `requestAnimationFrame`, so a non-compositing surface freezes it at 100×100

Dockview sizes itself from a `ResizeObserver` whose callback is deferred through rAF. In any browser surface that is not painting — a hidden preview pane, a background tab, a devtools-detached window — rAF never fires, so the grid keeps its initial 100×100 size, every panel is 100px wide, and every sash reports `dv-disabled`. It looks exactly like a broken layout or a missing CSS import. **Fix/policy:** before debugging dockview sizing, confirm the surface is actually compositing; verify in a real window or in headless Chromium (which does paint). Nothing about the CSS or the container needs changing. _[earned 2026-08-11, dockview 8.0.0]_

### UG2: dockview-react 8 logs a console error about `dockview-enterprise` on every load

`DockviewReact` unconditionally passes a `createContextMenuItemComponent` framework option, and dockview-core logs an error when the matching enterprise module is not registered. It appears once per page load, is not caused by anything the consumer does, and cannot be switched off through props. **Fix/policy:** ignore it, do not install `dockview-enterprise` to silence it, and do not write a "no console errors" assertion into the browser suite — that assertion would be permanently red for an upstream cosmetic. _[earned 2026-08-11, dockview-react 8.0.0]_

### UG11: The middle of a dockview tab is a dead line for drops, and a font change is what finds it

A tab is a drop target split 50/50 — left half means "insert before this tab", right half "insert after" — and dockview's own comparison is `x < 50%` on one side and `x > 50%` on the other, with no `center` zone registered for a tab. **A drop exactly on 50% resolves to neither, no overlay is shown, and the drop is discarded in silence.** No error, no console warning; the panel simply springs back.

Nobody would find this by hand, because a hand never lands on one sub-pixel column. A test does: Playwright's `dragTo` aims at the element's centre by default, which is that line. Whether it rounds to one side or lands on it depends on the tab's fractional width — **so the browser suite's re-dock test was passing on the width of the old font, and changing the tab's typeface (U32) turned it red with no logic touched anywhere.** Half a day went into bisecting a stylesheet for it.

**Fix:** aim the drop off-centre — `dragTo(target, { targetPosition: { x: 20, y: 12 } })` — and say in the test why. The rule generalises past dockview: **a drag test must never aim at the exact middle of a target that is divided in the middle.** A boundary is not a location; it is where two behaviours meet, and a test standing on it is measuring rounding.

Worth knowing as product behaviour too: a human dropping a tab precisely on another tab's centre line gets nothing and drags again. It is a real dead zone, one column wide, and not worth patching around. _[earned 2026-08-13, dockview-core 8.0.0]_

### UG12: A dev server that opens the browser as it starts races the backend behind its own proxy

Starting the editor with Vite's `server.open` means the page loads while the sidecar is still sweeping `.meta` files. Its first call is proxied to a port nothing is listening on yet, and the terminal prints `[vite] http proxy error: / … ECONNREFUSED` above the banner that says everything started. The editor then connects on its own — it has a connecting state and retries — so what the human is left with is **an error message about a working editor**, which is the worst kind to leave in a launcher: it teaches them that the red text in this window means nothing.

**Fix:** start the server with `open: false`, bring the backend up, and call `server.openBrowser()` afterwards. It opens regardless of the configured `open` value — that setting only supplies a path when it is a string — so the ordering costs nothing.

**The general form:** anything that opens a window is the *last* step of starting, after everything that window will immediately ask for is answering. The race is invisible on a fast machine and on an empty project, which is exactly why it survives until a real folder is opened on a real morning. _[earned 2026-08-14, Vite 8.2.1]_

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
- `kernel-2d/editor/panels/AssetsPanel.tsx` — the folder mirror, the worked example of a panel with a body, U22 (the control that puts a file in a human's project) and U29 (the one that renames, moves and deletes one).
- `kernel-2d/editor/shell/useFileMoves.ts` — the whole of a rename as a gesture: flush, plan, refuse, move, rewrite, report, and U30's remapping of what the human was looking at. The comment on why none of it goes through `edit` is the load-bearing one.
- `kernel-2d/editor/shell/references.ts` — which documents point at a file, and the same documents with its new path written in. Only `path` moves; `id` never does.
- `kernel-2d/editor/store/file-disk.ts` — the editor's two file operations, beside `document-disk.ts` and `meta-disk.ts`.
- `kernel-2d/editor/panels/InspectorPanel.tsx` — the worked example of U10, U11 and U12: every state it can be in has a sentence, and the editable one is reached only from the store. Three document formats have a body of their own now, each reached by what the document says it is.
- `kernel-2d/editor/panels/ProjectInspector.tsx` — U28: the third inspector body, the startup-level picker that reads a file before it writes a path, and the sentence that says what the current choice resolves to.
- `kernel-2d/editor/shell/zoom.ts` — what is left of the zoom controls after the ladder moved into the shipping layer (`editor-kernel` D20): stepping, and the wording.
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
- `kernel-2d/editor/shell/useSceneGestures.ts` — U20, U21 and U31: left-press to pick, place or stamp, middle-drag, space-drag, wheel-to-zoom, the framing keys, Esc, and the order they take priority in.
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
- `kernel-2d/editor/shell/usePlacePrefab.ts` — U24 as built: one gesture, reachable from the prefab and from what it just placed. Two doors rather than one optional argument, and the note saying why a button's `onClick` is what forces that.
- `kernel-2d/editor/shell/snap.ts` — U31's arithmetic, and the paragraph on why a grid needs an offset as well as a step. No React in it, so the properties are unit-testable.
- `kernel-2d/editor/shell/placing.tsx` — U31's state: what a press lands on and what it puts down, above the layout, in neither the document nor any file.
- `kernel-2d/editor/panels/PlaceByClicking.tsx` — the switch, shared the moment the second inspector wanted it, and the note on why turning it on places nothing.
- `kernel-2d/editor/panels/TexturePicker.tsx` — one picker, two owners, and the D5 round trip that fetches a texture's id. Lifted out the moment a prefab needed the same control as an entity.
- `kernel-2d/editor/shell/project-context.tsx` — U9. One folder read and one change stream per window.
- `kernel-2d/editor/shell/useAssetMeta.ts` — one file's import settings, re-asked when the folder changes as well as when the selection does.
- `kernel-2d/editor/shell/useProjectTree.ts` — the folder, kept current: the re-read of U5, the settle of U6, and the stale state of U7.
- `kernel-2d/editor/shell/App.tsx` — the shell: status strip above, docking layout below, providers around both.
- `kernel-2d/editor/shell/useSidecarStatus.ts` — how the editor learns which project it is connected to, and what it does when the answer stops coming (U3).
- `kernel-2d/editor/shell/StatusStrip.tsx` — the connection line, including the `data-testid` hooks the browser suite reads.
- `kernel-2d/editor/shell/shell.css` — the frame around the docking layout, and **every colour the editor has**: the palette tokens, what the accent is allowed to mean, and the block that repaints dockview's theme in those same tokens (U32).
- `kernel-2d/vite.config.ts` — the editor's root folder, the `/api` proxy to the sidecar (U2), and the loopback binding (UG3).
- `kernel-2d/scripts/editor-server.ts` — the editor window's host, port, and open-a-browser knobs, with their environment variable names.
- `kernel-2d/tsconfig.editor.json` and `kernel-2d/tsconfig.base.json` — the browser half of U4.

- `kernel-2d/editor/store/open-documents.ts` — the one document store per window and the hooks panels read it through. The API itself is `editor-kernel`'s (D7); this is where the UI meets it, and where editing is suspended while a level runs (U26).
- `kernel-2d/editor/shell/play-mode.tsx` — play as window state: what it is, what pressing Play does before it reads the file, and why it never touches one.
- `kernel-2d/editor/shell/panels.tsx` — every panel declared once, and the wrapper that puts all but the Viewport out of reach while a level runs (U26).
- `kernel-2d/editor/shell/scene-view-context.tsx` — the scene renderer's context, and `useDrawScene`: the request key that distinguishes two derivations of one level, and the answer to U27.

**Not yet written** — until a path appears here, the contract does not exist and must not be assumed:

- Inspector auto-generation from Zod schemas. There are two hand-written inspectors now, which is the point at which generalising has something to generalise *from*.
- Gizmos for rotate and scale, a marquee, dragging a pivot or a frame grid, and cycling through overlapping sprites with repeated clicks. Position is the one thing a viewport can change; everything else is the Inspector's, deliberately.
- A grid or rulers *drawn*. The snap is settable (U31) and nothing renders it, so a board lines up to lines nobody can see. The overlay that would draw them is U16-shaped work and has not been asked for.
- Painting by dragging: a stroke that stamps one copy per cell crossed. Clicking is quick enough, and a stroke is the first step toward a brush, a rectangle tool and an eraser — none of which the kernel has been asked for and all of which are what `genre-spinup` S1 sends at a game folder.
- Dragging inside the Hierarchy tree, and nesting. The list is flat and reordering is two buttons.
- Multi-selection. `Selected` is a union of one thing; making it a list is a change to that type and to every reader of it. The rename control operates on one file for the same reason.
- Dragging a file inside the Assets tree to move it. Moving is a name field, a folder chooser and a button (U29); drag-and-drop over a tree is a second gesture surface for an operation that already has one.
- Deleting a folder. Renaming and moving take folders; deleting is one file at a time (`editor-kernel` D22).
- A panel menu or anything that lists the viewport's shortcuts. `Home` and `F` are named in the caption's own sentences and on two buttons, and nowhere else.
- Two scenes open at once. `openScene` is one path.
- The panel menu and saved layouts (UG4).
- A keyboard shortcut registry. There is exactly one handler (U13), and one does not need a registry.
- Anything that shows the undo stack — a history panel, an Edit menu, an "Undo <label>" caption. The labels exist and `peekUndo`/`peekRedo` expose them; nothing reads them yet.
- Any control over a level while it is running: a camera that can be moved, a pause, a step, a frame counter, or editing that the running level picks up. Play inherits the editing camera and freezes it, and the rest of the editor goes read-only until Stop.
