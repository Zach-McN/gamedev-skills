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

**Amended again when selection went plural. The entity case holds a list; nothing else about U8 moved.**

```ts
| { kind: 'entity'; scene: string; entities: readonly string[] }  // never empty
```

The plural lives *inside* the union rather than beside it: a second "and also these" field would be a second answer to "what is selected", which is the pair problem the union was adopted to avoid. One scene rather than one per entity, for the same reason — a selection spanning two levels is not something the editor can act on, so it is not something the type can spell.

Two invariants carry the whole change, and both are the same move as the union itself — *a state that cannot be written down is a state nobody has to check for*:

1. **Never empty.** An empty entity-selection and `{ kind: 'none' }` would be two spellings of one state. Removing the last entity lands on `none`, enforced in the one function that can break it.
2. **The last element is the primary one** — the most recently touched. Everything singular in the editor (the Inspector, `F`, `G`, Frame selected, the reorder arrows, Duplicate) reads a `selectedEntity` convenience that returns it. **This is what made the change cheap:** going plural touched the selection context, the two panels that build a selection, and the overlay that draws it — and no other panel grew a new thought, because none of them had to learn what a list means.

The request that prompted it asked for selections to go on the Ctrl-Z stack. **They did not, and the reason is worth recording because it will be asked again:** U8's guarantee is the whole of why Ctrl-Z is predictable, and a stack holding both edits and selections makes every press a coin toss between reversing work and reversing a click. What the asker actually wanted was underneath it — deleting six entities should be *one* press, not six — and that is a property of the **edit** being one transaction, not of selection being undoable. When a request seems to want selection in the history, look for the multi-target edit it is really about. _[earned 2026-08-15]_

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

**Amended 2026-08-14: the right *click* is claimed; the right *drag* still is not.** A right-click asks about what is under it (U39's window), doubles as the way out of a placing mode, and the browser's context menu is suppressed on the surface unconditionally — including while a level runs, when the right button is the game's. A click is not a drag: the flythrough idiom stays unclaimed.

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

### U33: A modal gesture is started by a key, owns the surface while it runs, and has three ways out

Blender's `G` arrived in the viewport: press it and the selected entity moves with the pointer, with no button held; `X` and `Y` hold it to one axis from where it started; a click puts it down; `Esc` puts it back. It is worth having for a reason about hands rather than about parity — **the sprite you are placing is usually the one your cursor is covering**, so a gesture that must start on the sprite starts by hiding it, and one that can start anywhere does not. The same argument is why it wants no button held: the hand is free to travel the whole panel.

**The origin may not exist yet, and that is the load-bearing detail.** A move is travel from where the pointer was when the gesture began (U21), and when the key is pressed the pointer may be over the Inspector, or outside the window, or somewhere it has not been since the last level was open. So the origin is *null until the pointer is next seen over the picture*, and the first sighting is taken as the origin rather than as a movement. The companion rule: **forget the last-known pointer position when it leaves the surface**, or a grab started with the hand elsewhere measures from a point the cursor left ten minutes ago and throws the entity across the level. A grab already running keeps its origin when the pointer leaves — that is what travel means, and the hand is allowed to come back.

**While it runs it owns the surface, and every exception is a jump.** A press cannot select or pan, the wheel cannot zoom, the framing keys cannot move the camera. Each of those either changes what the travel is measured against (the camera's scale is what turns pixels into level units) or changes what is being moved, and both show up as the sprite leaving the cursor. This is U21's "one gesture layer decides what a press means" applied to a gesture that has no press: the mode is asked about first, above space.

Five things that only appear once it is built:

1. **It lives in a ref, not in state.** The keyboard starts it and the pointer drives it, so a grab read from React state is one render behind on the first movement after the key — and the entity jumps by however far the pointer travelled in that frame. The state is the copy that gets *drawn*.
2. **An axis lock zeroes the travel on the other axis** rather than remembering a second position, which is what makes "from where it started" true by construction. Apply it on the keypress rather than waiting for the next movement: the human pressed `X` because they want the vertical gone *now*, and a lock that took effect on the next wobble reads as not having worked. A second press of the same key frees it — Blender spends that press on local axes, and a 2D entity has none to offer.
3. **There are three ways out, and the third is a list.** Put it down, call it off — and *taken away*: the window losing focus, the level closing, Play starting, another panel moving the selection. All of those cancel rather than drop, because nothing was decided. Miss them and a grab keeps running invisibly, and the next mouse movement over the picture moves an entity nobody is moving.
4. **The caption is the only place these keys exist.** A mode with nothing held has no affordance at all — no button is down, no handle is being dragged — so the bar names what is moving, which axis it is held to, and the way out, in as few words as the bar can show without clipping, with the whole sentence in the tooltip (U10, UG8). The lock also gets a line drawn across the picture through the entity's *position*, not the middle of its outline: a sentence is read once, a line is still there while the hand is moving.
5. **The key that copies had to share an implementation with the button that copies.** `Shift-D` duplicates and selects the copy — the Hierarchy already had that button, so the operation moved into a hook and both call it. Two implementations of "what a copy is" would differ the first time the format grew a field. Same shape as `usePlacePrefab` (U24), for the same reason.

**What made this small: it wrote no undo code.** A grab is the drag's own machinery with a different trigger, and calling one off is `editor-kernel` D7's `abandonEdits` — a run identity plus the inverse patches already recorded. The gesture layer grew a mode, the document layer grew one primitive, and nothing in between had an opinion. _[earned 2026-08-14]_

**Confirmed as a shape by the second one — `R` to rotate — which reused all five points above unchanged and added three that only a second modal gesture can show.** Building it was mostly transcription, which is the evidence that the five were the shape rather than facts about grabbing.

6. **"One modal gesture at a time" becomes a rule the moment there are two.** With one, nothing had to say it; with two, "the selection is being moved *and* turned, by two gestures measuring from two origins" is a reachable state and a meaningless one. Each start refuses while the other runs, each key-block returns for everything it does not handle, and a press finishes whichever is running rather than starting anything. The cheap mistake is guarding only the new one.
7. **A gesture measuring an *angle* carries two silent sign traps a translation does not.** Screen y counts down and scene rotation is counter-clockwise in a y-up world, so the measured bearing is negated exactly once — and neither zero nor two negations crashes anything, they just turn everything the wrong way, which reads as the feature being backwards rather than as arithmetic. And `atan2` **on** the pivot is zero by convention rather than by meaning, so a pointer near the middle swings the angle through tens of degrees per pixel of noise: a dead radius is required, not defensive. Put the flip in one named function and unit-test the sign on its own; a browser test can only sample a rotation, and a reversed one looks exactly like a correct one dragged the other way.
8. **Rotating a multi-selection is orbit *and* spin, and testing only positions passes half of it.** Each entity swings about the group's pivot and turns on its own axis; doing only the first slides sprites round a circle while they stay upright. Both belong in one assertion. The pivot is the **mean of the positions**, not the centre of the bounding box — two sprites of very different sizes have a box centred nearer the big one, which is not what "turn these around their middle" means. One entity then needs no special case: the mean of one position is that position, so it turns in place by the same arithmetic.

**Extending the move to a multi-selection needed no ninth point, and one new rule.** The gesture layer did not change: `moveBy` still takes the entity under the cursor, and the group is remembered beside it. What is worth recording is where the *group* comes from and what a press means:

- **The snap is applied to the anchor and everything else carries the same travel.** Snapping each entity on its own account pulls sprites three units apart onto one grid position — a formation destroyed by being nudged. Same argument as not snapping the positions a rotation orbits, and it generalises: **in a group transform, exactly one member is snapped and the rest inherit the delta.**
- **A press inside the selection must not collapse it, and a press outside must replace it** — otherwise a group can never be picked up. The set is decided from the selection *as it was before the press*, which is also the only reading available: the press calls `select` and then `beginMove`, with no render in between, so the selection object in hand is the old one either way. A rule that wants the pre-press value is the rule to write when the state is one render behind (`UG5`'s shape, used deliberately rather than worked around).
- **The click that collapses a group is decided on the release**, because until the pointer travels nobody knows whether a press was a click or the start of a drag. Keeping only the first half leaves a multi-selection with no way out except clicking empty space, which reads as the selection being stuck.

Two smaller things worth carrying: the angle is **measured from the start bearing, never accumulated** (the drag's own rule, and what makes a full circle exactly the identity — the cheapest test for someone changing it later); and the gesture publishes the pivot and the pointer **in the space the overlay draws in**, so the gizmo cannot disagree with what is being rotated about, and so a browser test can ask the editor where the pivot is instead of guessing it from one sprite's outline. _[extended 2026-08-15]_

**The third — `S` to scale — was transcription plus three points, and the first of them is the one worth having.** Points 1–8 were reused unchanged; that it took an afternoon is the evidence that the shape is a shape.

9. **The axis lock does not mean the same thing in every gesture, and nobody notices until the third one.** A grab's `X` is the *level's* axis — travel along world x. A scale's `X` is the **sprite's own**, and it costs nothing to be local: a transform scales before it rotates, so multiplying `scaleX` stretches the sprite along its own side however it is turned, with no rotation read anywhere. The same key, in the same hand position, is world-space in one gesture and local-space in the next — correct rather than sloppy, because it is what each gesture's number *is*. What has no local answer is the **group** half: several entities have several locals, so positions spread on world axes while shapes stretch on their own. For one entity the two coincide exactly (its offset from its own pivot is zero), which is why this ships without a world/local toggle and nobody meets the gap.
10. **Express the lock as a factor pair, not a branch.** `{x: f, y: f}` unlocked, `{x: f, y: 1}` for X. As a branch, "which axis" leaks into the arithmetic, the transaction, the gizmo and the caption; as a pair, exactly one place — where the key is read — knows an axis exists. Rotate never had to learn this, because a flat level has one axis to turn about.
11. **Every group transform has two halves, and the obvious half passes half the tests.** Rotate is orbit-and-spin; scale is spread-and-grow; a move is the degenerate case with one half. An implementation that only grows the sprites passes every assertion about sizes. Write the pair assertion first — it is the same test one gesture over, and it is the one that fails for the bug that actually happens.

Two smaller ones. A scale's number is a **ratio of two on-screen distances**, which makes the dead radius load-bearing for a second reason — dividing by a reach of nothing, not only noisy bearings — and makes blocking the wheel mid-gesture non-negotiable, since the wheel changes the denominator with the hand perfectly still. And **a gizmo has to make its own number legible in its own way**: a swept angle gets an arc, but a ratio has nothing to sweep, so the readable thing is the *starting reach marked on the line* — which side of that mark the cursor is on is the whole of "bigger or smaller", and it is what makes a snapped factor read as steps rather than as a laggy mouse. _[extended 2026-08-15]_

### U34: A second way of looking at the same folder is two components either side of one bar, and where you are browsing is window state

The Assets panel gained the file-explorer view: a grid of tiles showing one folder at a time, a breadcrumb, and a cog offering tree / icons / both-at-once. Four things came out of it, and the first is the one that shapes the rest.

**Which view, and which folder, live above the docking layout** — beside selection, the camera and the placing settings (U8, U19, U31). Not because two panels need them, but because dockview unmounts a panel body when its tab is dragged (U9), so a view the human chose and a folder they had walked three levels into would be thrown away by moving the panel. It is also what lets `useFileMoves` follow a rename into it, which U30 requires the moment any window state is keyed on a path — and the open set of the tree is keyed on a path too, one entry per open folder.

**Which folder the grid is in and which folders the tree has open are two pieces of state, not one.** A tree has six folders open at once; a grid is inside exactly one. Deriving either from the other means walking into a folder in the grid shuts five others in the tree, or opening a second folder in the tree moves the grid somewhere nobody asked. What they *do* share is a direction: entering a folder anywhere opens the way down to it in the tree, so the two halves of the split view never disagree about where you are.

**The breadcrumb is on screen only when the grid is.** A tree has no current folder, so a breadcrumb over one is a sentence about somewhere the human is not — worse than no sentence (U10 pointed the other way: the failure mode of a caption is not only saying nothing, it is saying something untrue). The bar itself stays in every view, because the cog is how you get back.

**One press selects, two enters, and both halves ask `asset-rows.ts`.** The explorer's own split, and there is no improving on it: selecting is what the Inspector answers about, and a folder has to be selectable without being walked into. Both views get their rows from the same place (`editor-kernel` D4), so a `.meta` folds into its file's tile exactly as it folds into its row; two rules for what a folder contains would disagree the first time either changed, and the symptom is a file that exists in one view and not the other.

What the view is *not*: not saved to disk, not in `project.json`, not per-project, and not in a document. A reload is back to the default view with everything shut. (The default was the tree when this was built; since later on 2026-08-14 it is the icon view, the tiles being the view a human parses fastest and the tree one cog press away.) _[earned 2026-08-14]_

### U35: A drag between two panels uses the browser's own drag for the *gesture* and the window's own state for the *payload*

Dragging a file out of the Assets panel and letting it go over the level is the third way something gets placed, after a button and a mode. The split that made it small is the one worth copying.

**The browser's drag-and-drop is used for everything it is good at:** it draws the ghost of the row being carried, it sets the cursor, it fires `dragenter`/`dragover`/`drop` with client coordinates, and it crosses from one docked panel into another without either of them arranging anything. A pointer gesture of our own would have re-implemented all of that and collided with the viewport's existing press handling (U21).

**What it is *not* used for is carrying the path.** A `dataTransfer`'s *data* cannot be read until the drop — only its *types* are visible during `dragover` — so a viewport that wants to say "Drop enemy-slime.json here" while the pointer is still moving cannot get the name from there. Both ends are in one window, so the path lives in the window's own state beside the other placing settings (U31), and the `dataTransfer` carries one marker type saying "this is one of ours". One copy of the fact, and the marker is what keeps a text selection or a file dragged in from Explorer from being treated as an asset.

**What a dropped file *is* is decided by reading it, at the drop.** The panel cannot know: a `.json` is a level or a prefab according to what it says inside itself (U11), and a `.png`'s extension is a guess its own `.meta` may overrule. So every file is draggable and the drop reads — a `.json` through `/api/document` for its `format`, anything else through `/api/meta`, which answers *what it is* and *the id to point at* in one round trip (the id being needed anyway, D5). This is U28's rule — validate by reading the file, not by trusting the path — arriving a second time, which is what makes it a pattern rather than a one-off.

**A file that cannot be placed gets a sentence, not a row that refuses to be picked up.** "level-02.json is a level, so it cannot be placed in a level" and "jump.wav is a sound" are answers; a row that silently declines to move is a bug report waiting to be filed (U10). Only folders are undraggable, and they say so with `draggable={false}` rather than by omission — a fact written down is testable, an absence is an oversight.

Three smaller things, each of which was a wrong version first:

1. **`dragover` must cancel the event on every move, not just on `dragenter`.** Otherwise the browser refuses the drop, the ghost springs back to the panel, and nothing is said anywhere.
2. **`dragleave` fires when the pointer crosses onto a child**, and the canvas is a child — so the naive handler switches the highlight off in the middle of the picture. Count enters and leaves rather than comparing `relatedTarget`; the count stays right whatever the panel gains later.
3. **The highlight may not change the box.** It is an inset ring, because the stage's size is what framing is computed against (UG8).

**A drop selects what it made** — unlike repeat-placing, which must not (U31). One drop is one deliberate act and the human's next move is to tune the thing they just put down. Selecting still happens outside the transaction (U8), so one drop is exactly one press of Ctrl-Z, and the feature contains no undo code.

**The third caller is what forced the recipe out of the hook.** `usePlacePrefab` reads its prefab from the document store; a dropped prefab has never been in the store, because reading it off disk is what identified it. So what an instance *is* moved into a plain function both call — the same shape as `useDuplicateEntity` (U33), and for the same reason: two copies of "what a placed instance is" is two chances to write the one that copies the prefab's components instead of pointing at them. _[earned 2026-08-14]_

### U36: Where a browser is, is a trail rather than a folder — and the pane sizes it sits in are fractions rather than widths

Two small things the Assets panel wanted once it was a file browser, and each has a general form worth keeping.

**The mouse's side buttons hop along a trail, and a trail is not the path.** Back from `assets/textures/characters` after one jump out to `assets` lands three deep again, where the human actually was; `parentOf` would land at the top of the project. So the browsing state is `{ folders: string[], at: number }` rather than one path, and it behaves the way a browser's does — including the part that decides the shape: **going somewhere new after stepping back drops everything ahead**, because a forward into a folder you have since changed your mind about is not a place you were on the way to. Everything already true of one path is now true of the list: U30's rename hook remaps every entry, or stepping back lands on a folder under its old name.

**A resizable pane stores a fraction, never a width.** A docked panel is resized constantly, so a divider that remembered pixels leaves one pane the same size in a panel twice as wide, which is not the split anybody chose. Clamp in the state rather than at the handle — no caller may leave a pane too narrow to take hold of again — and give the pane a `min-width` as well, which loses to `flex-basis` going up and wins going down, exactly the asymmetry wanted. Two sizes for the handle itself: a hairline for the eye, a wider strip for the hand. And **double-click resets it**, because a divider is the one control that can be dragged into a state with no affordance left to explain itself, so the way out has to be on the thing that got you there.

Both are window state above the layout, beside the view and the folder (U34): never serialized, never in a document, gone on reload. _[earned 2026-08-14]_

### U37: Dragging a row to reorder is a second door on the arrows' edit, and the two drag surfaces are deaf to each other by marker type

The scene panel was renamed **Outliner** (it was Hierarchy until 2026-08-14 — entries above that say Hierarchy mean this panel), and its rows became draggable as an alternative to the `↑` `↓` buttons. The shape that kept it small:

**One implementation of the reorder, two ways to ask for it** — the same rule as Duplicate and `Shift-D` (U33). The drag ends in the identical `'Reorder entity'` transaction the arrows use, so one drag is one press of Ctrl-Z and no undo code was written. What the drag adds is only the question "which slot is under the pointer": a row is divided at its midline into before and after, below the last row means the end, and **the slot is answered by one function asked on every `dragover` (for the line between rows) and again on the drop (for the edit)** — two readings of one rule, so the promise and the act cannot disagree about where "here" is.

**A slot where letting go would change nothing answers null.** No line is drawn there — a line is a promise that something will happen — and no transaction is opened, which the store would discard anyway (`editor-kernel` D7 drops empty patch sets; the guard here is so the *line* tells the truth, not to protect undo).

**The row drag wears a marker type of its own** (`application/x-kernel-2d-entity-row`, beside U35's asset type), and each surface's `dragover` answers only to its own marker. A row carried over the picture places nothing and a file carried over the list reorders nothing, by construction rather than by checks at the drop — the same reasoning that keeps an OS file drag inert (U35).

Two practicalities:

1. **The dragged row's id rides in a ref, never in state.** Nothing needs to *render* from "what is being dragged" — only "where it would land" is drawn — and setting state inside `dragstart` re-renders the source row at the exact moment Chromium decides whether to begin the drag, which is the documented way an HTML5 drag gets silently cancelled. Chosen up front on that knowledge rather than earned from the failure here.
2. **The insertion line is drawn outside the flow** — an absolutely positioned edge on the row, `pointer-events: none` — so it cannot move the rows it sits between (a target that shifts under the gesture aiming at it is UG13's bug) and cannot fire `dragleave` flicker of its own. It wears the accent for the drop-ring's reason (U32): the most current thing on screen for as long as it is there.

The test aims at the quarter-heights of a row, never its middle: the midline is the before/after boundary, and a boundary is not a location (UG11). _[earned 2026-08-14]_

### U38: The panel menu lives on the one bar that cannot be closed, and it is a third derivation of the panel declaration

The Windows menu — a button on the status strip listing every panel, a tick beside the ones on screen — is UG4's missing door. Four decisions in it:

1. **It lives on the status strip because the strip is not a panel.** A "reopen a panel" control inside any panel is UG4 one level up: close that panel and the way back is gone with it. The strip is the one piece of the window that cannot be closed, which makes it the only correct home, not merely a convenient one.
2. **The list is `PANELS` itself** — the same declaration that already produces the component registry and the opening layout (U1). A panel added to that file appears in the menu with no second list to forget. This is the third thing derived from the declaration, which is the payoff U1 promised.
3. **Picking a panel is focus-or-spawn (`summon`), and a respawned panel lands as a tab in the active group** rather than at its old spot. The spot may have been closed along with it, and a tab the human can drag anywhere beats a guess about where it used to be. `bringToFront` stays separate and non-spawning: a *selection* wanting a tab forward is not the human asking for a closed panel back, and a layout that reopens panels because something was clicked is the layout changing itself unasked.
4. **The dropdown copies the cog's mechanics** (U34's view menu): Escape handled on the menu's own subtree and stopped — the viewport owns Escape for calling off a grab (U33), and whichever thing is open should be the one that hears it — plus a window-level `pointerdown` that closes on a press anywhere else. Two of these menus now exist as two copies; a third wanting the same shell is the moment to lift it, not before (U28's restraint).

Respawning a viewport-shaped panel is safe *because of* U15/U18: the renderer's canvas is held above the layout and adopted on mount, so closing and reopening the tab is the same teardown a tab drag already was, and no WebGL context is spent. _[earned 2026-08-14]_

### U39: The right-click window is the Inspector's field in a second place, and an overlay on a native-listener surface defends itself natively

A right-click on an entity opens a small window beside it holding the entity's position — the first editable settings in the picture itself, with more to come. Four decisions:

1. **The right press is one more rule in the one gesture layer (U21), slotted into the existing priority order.** A grab swallows it (Esc is the way out of a grab); space does *not* claim it (a right-click while space is held is still a right-click, because only left and middle pan); a placing mode makes it the way out of the mode — the second button's meaning in every editor with a placement mode — and only then does it ask about what is under it. Empty space closes whatever is open, which is the same press pointed away.
2. **The window is the same field as the Inspector's, not a copy of it.** Same transaction API, same merge key (`path#id#x`), so a value typed in either appears in the other, typing in both within the merge window is one undo step, and there is no second implementation to drift. The window also *selects* what it opened on, so the outline, the Inspector and the window all describe one entity.
3. **It is anchored in screen space, so its ways-out list includes the camera.** U33's list-of-ways-out shape again: Esc (focus handed back to the picture), a press elsewhere, right-click on nothing, the selection moving, the entity going, a drag or grab starting, Play starting — and any pan or zoom, because a window anchored to pixels points at nothing once the camera moves. The close-on-camera-change is a snapshot of the camera taken at open, compared in an effect.
4. **Presses inside the window are stopped with native listeners, not React handlers.** The gesture surface listens with `addEventListener`, and a native listener on an ancestor runs before React's delegated handlers do — so a React `onPointerDown` calling `stopPropagation` in the overlay fires *after* the surface has already treated the press as a pick, deselected the entity, and closed the window under the cursor. The overlay attaches its own native `pointerdown`/`mousedown`/`wheel` stoppers in an effect. (Chosen up front from how React delegation works, and held in place by the test that clicks a field and asserts the window survives.) Drags the surface has *captured* are unaffected: pointer capture retargets events to the surface directly, so a sprite dragged across the window keeps its gesture.

The browser's own context menu is prevented on the surface unconditionally — not gated on the editor being enabled, because while a level runs the right button is the game's, and the browser menu over a running game is the same wrong answer. _[earned 2026-08-14]_

**Filling it out — rename, Frame, Duplicate, Delete beside the position — added four points, and the first is the one that decides what such a window is for.**

5. **A context window's justification is *distance*, so what goes in it is chosen by what a hand reaches for while the cursor is already there** — not by what would fit, and not by "everything the Inspector has". Every verb added already existed somewhere (the Inspector's fields, the Outliner's toolbar, the viewport's `F`); none of it is new capability, and that is the point. The test to apply to a candidate is "would somebody cross the window for this, mid-gesture?" — which admits Delete and excludes, say, the texture picker, whose own panel is where you are going anyway once you start picking.
6. **A window that selects what it opens on can borrow the selection-shaped actions without owning them.** Delete acts on *the selection*, and opening the window makes the selection exactly this entity — both doors do it, and the window closes the instant the selection moves off it, so "the selection" and "this entity" cannot come apart while it is on screen. So it calls the shared `useDeleteEntities` unchanged rather than growing a delete-just-this-one. **The assertion that makes it safe is the one about a multi-selection**: right-click one sprite of six and press Delete, and the other five must survive. Without it, a plausible implementation deletes all six and the test suite is silent.
7. **Buttons that change the selection or the camera need no closing code — the window's existing ways-out already cover them, and letting them do it is the point.** Duplicate selects the copy (selection moved → closed); Delete leaves no entity (entity gone → closed); Frame moves the camera, which closes it at the *screen-anchored* door and not at the list-anchored one. Writing `onClose()` into the buttons would be the component having an opinion about two panels' anchors, and would close the list's window for a reason that is only true of the picture's. Asymmetry that falls out of a rule is a feature; asymmetry written by hand is a bug waiting.
8. **Growing the card makes placement a flip, not a clamp — and a flipped card is pinned by its *bottom*.** At 100 pixels tall, sliding a card up to fit inside its panel keeps it near the press; at 130, in a 200-pixel panel, it ends up over a hundred pixels from the thing it is about. So the shared placement puts the card **below the press if it fits, above it if it fits there instead, and clamps only as a last resort** — what every context menu does.

   Which edge a flipped card is pinned by is the part that is easy to get wrong, and was, twice in one session. Pinned by its top at `press − height`, placement depends on `height` — and `height` is a **reservation written by hand** beside a component that grows when it has a path to preview or a refusal to show. Reserve too much and the card opens with a visible gap under the press (a real 47-pixel one, caught by a test); too little and it opens over it; and whatever it grows afterwards grows *downward*, into the edge it was moved away from. Pinned by its bottom, the card touches the press whatever it turns out to measure, growth goes upward into the room that made the flip the right choice, and the reservation is only ever used to decide *which side* — a judgement that survives being a few pixels out. **A card that can change size is anchored on the edge nearest the press.**

   It also invalidates the obvious browser assertion. "The card's top-left is near the click" is false by design once it flips, so the test fails for the behaviour improving. Assert the **gap from the press to the nearest edge of the card**: true of all three placements, zero when the card covers the press, and shared by both cards' specs for the same reason the placement itself is. _[extended 2026-08-15]_

**The first thing in it that is *not* somewhere else — Snap to grid — tested point 5 from the other side, and added three rules about buttons.**

9. **New capability is admissible in a context window when the window is the only place its input is on screen.** Point 5's test ("would somebody cross the window for this?") is about relocating existing verbs and says nothing about new ones. The one that earned its place puts *this* entity on the grid, and it belongs here because what it acts on — a position — is the two fields directly above it, and because the alternative is working out a rounded number and typing it. It sits below the line separating fields from verbs (it *happens* when pressed) and above the three relocated ones, nearest what it changes.
10. **A command that would change nothing is disabled and says why, rather than being safe and silent.** The transaction store already drops an edit that produces no patches, so the press would be harmless — and harmless is the problem: a button that appears to work and does nothing reads as broken. Greyed, with a tooltip saying "already on the grid — every 16 from 8", it has answered the question instead. The shape generalises to any tidy-this-up verb: **the no-op case is the one worth spending a state on.**
11. **A control reachable from a panel that does not show the setting it uses must name that setting where it is.** The window opens over the picture, where the interval and offset are in the bar underneath, *and* over the Outliner, where they are not on screen at all — so "Snap to grid" alone is a button whose result cannot be predicted from the door it was reached through. The numbers go in the tooltip, with the fact that the grid still applies while the switch is off. **A shared component with two doors has to be legible from the worse door.** _[extended 2026-08-15]_

### U40: A level's music is the scene's own field, chosen in the scene's inspector body, and only a run ever plays it

The first sound in the editor, and three decisions that kept it small:

1. **The field is on the scene document, not on an entity.** Which sound a level plays is a fact about the level — one level, one track — and an entity carrying it would make "which entity holds the music" a question with fifty wrong answers. The scene's inspector body (the U28 fall-through that had nothing editable) gained its first control: a picker offering every audio file in the project, writing a D5 reference through the transaction API, "Nothing" deleting the field rather than emptying it. Editable only while the scene is the open one, because a control must edit the store (U12) and opening is what puts it there.
2. **Starting and stopping the music belongs to the run, in the same effect that starts and stops the level** (`running-level.ts` behind Play, `start-game.ts` in a shipped folder). Editing is silent because nothing else ever asks — a property of who calls, not a mute flag anywhere. Travelling through a door switches to the new level's track, or to silence when it has none, because each `begin` states its own music.
3. **The playing state is a data attribute read back off the sound system** (`data-play-music`, `phaser4-runtime` P8/P4), refreshed by the ten-a-second description a running level already publishes — which is how a browser test asserts audio without hearing anything.

**Worth noticing: this is the third asynchronous asset picker** (texture, startup level, now music), and the second that fetches an id from a `.meta` — the music picker is the texture picker with a different type filter. U28 declined to lift at two; at three, with two of them near-identical, lifting the shared shell is now justified the way `place-into-scene` was at its third caller (U35). Proposed rather than performed, one feature per session. _[earned 2026-08-14]_

### U41: A modifier does not add a gesture — it adds an *answer* at the one place that already decides what a press means

Shift-click and Ctrl-click arrived in two surfaces at once (the Outliner's rows, the picture), and the tempting shape in both is a branch at the call site: an `onClick` that reads `event.shiftKey` and calls a different function. In the picture that shape is actively wrong, and the reason is already written on the wall of `useSceneGestures.ts` — **one element cannot have two opinions about one `pointerdown`**. The modifier is not a new gesture competing with pick-and-drag; it is a third answer to the question that hook exists to answer once.

So the press handler computes a `SelectMode` (`replace | add | remove`) and hands it to the same `select` it always called. Three things fell out of that, and each is a bug avoided rather than a nicety:

1. **Only `replace` starts a drag.** A modified press that also armed a move would drag one entity out of a set the human was still assembling. The gate is one line, in the place that already decides whether a press drags at all.
2. **A modified press on empty space does nothing** — it does not clear. A plain click on empty space still clears, so nothing is lost; but a six-click selection destroyed by a seventh that missed is what makes a multi-select feature the human stops using. **This is the acceptance criterion worth writing before the code**, because every positive test passes without it.
3. **Shift wins over Ctrl when both are held.** Arbitrary, but one of them must: doing nothing reads as a press that missed.

**The same three meanings must hold in both surfaces.** The list idiom for Shift is range-from-last-click, and it was declined: the picture has no order to take a range along, and a modifier meaning one thing in the list and another over the level is worse than the missing convenience. Where two surfaces share a modifier, the surface with fewer available meanings sets the vocabulary.

**And the key that acts on the result goes in the same hook, not in a listener of its own.** `Delete` needed four guards — not while typing, not during a grab, not while a level runs, not without a scene — and `useSceneGestures` had already answered all four for `Shift-D`, on a **window** listener (which is why `Shift-D` has always worked with the hand in the Outliner). A second window listener would have re-derived all four and then raced this one. The smell is general: *before adding a global key handler, check whether an existing one already has the guards it needs* — the guards, not the topic, decide where it lives. _[earned 2026-08-15]_

### U42: A mode and the modifier that overrides it are one exclusive-or, computed in one place — and the modifier means *the other thing*, not "off"

The snap gained a toggle, and the key that had meant "ignore the snap for this drag" became the key that **inverts** it. Those are not the same feature, and the difference is the whole entry: with snapping off, the modifier now puts the entity *on* the grid. That half is the one nobody thinks of and the one people actually use — lay a level out by eye, then hold the key for the one piece that must line up.

Three things follow, and each is a bug avoided:

1. **`setting.on !== held` lives in one function**, which every placement path calls — drag, keyboard grab, file dropped on the canvas, prefab stamped by a press. Spelled out per call site it is N chances to write the inversion backwards, and backwards is **silent**: things land on the grid when you asked for anywhere, which reads as the toggle being stuck rather than as a modifier being inverted. The payoff showed up immediately — switching the toggle off changed all four ways of placing something, including the two nobody re-tested.
2. **The geometry function must not consult the toggle.** `snapTo` asks only "where is the nearest grid position"; asking `on` there as well makes the inverted case *unreachable*, because it has to snap to a grid the toggle says is off. Worth a comment at the function saying so, because re-adding the check looks like tightening.
3. **Rename the boolean when its sense flips.** The argument was `free` while the old key meant free; it is `invert` now. A boolean whose meaning has inverted and whose name has not is the single most reliable way to make the next reader write case 1 backwards.

**Apply it on the keypress, not on the next movement.** Same argument the axis lock in this codebase already makes, and it bites harder here: a keyboard grab routinely sits with the pointer *completely still* while the eye decides, so a modifier that waited for a wobble reads as not having worked at all. That needs the gesture layer to remember the travel of the move in progress so a keypress can replay it under the new modifier — a small ref, and the thing that makes the feature feel instant in both gestures rather than only in the one where the hand is already moving.

**Check the modifier against every other reading of that key on the same surface.** `Ctrl` here also means "take this entity out of the selection" on a press. They coexist only because a modified press deliberately starts no drag, so the snap reading is reachable exclusively *inside* a move a plain press began. That is a fact to verify and write down, not to hope for. _[earned 2026-08-15]_

### U43: A control disabled as a side effect of a readiness signal flickers — disable it on the intent instead

Play was greyed while an entity was being moved, but only incidentally: the condition was "the picture on screen is a picture of the level as it is now", which is false on every mouse movement and **true again in every pause between them**. So the button flickered — worst of all during a keyboard grab, where the hand is often still for seconds and the move is nowhere near finished.

**Readiness is sampled; intent is not.** Any control gated on "has the async thing caught up" will strobe for exactly as long as the human is doing something intermittent, and no amount of debouncing fixes it, because the signal is *correct* — it is answering a different question. The fix is to add the intent as its own clause (`gesture === null`) and let the readiness clause keep meaning what it means.

Two tells that this is the bug you have: the control is correct on the frame the gesture starts (so any assertion made straight afterwards passes), and the complaint is about *flicker* rather than about the state being wrong. **The test therefore has to wait** — start the gesture, hold still well past the point the renderer has certainly caught up, and assert the control is still disabled. A version without the wait passes against the bug it was written for, which is worth checking by reverting the fix and watching the test fail. _[earned 2026-08-15]_

### U44: A floating window two panels can open is *one* window in shell state with an owner — not one component mounted twice

The right-click window (U39) had a second door added: a right-click on an entity's row in the Outliner opens the same window on the same entity. The obvious shape — the Outliner grows its own `useState` anchor and renders its own `EntityPopover` — is wrong for a reason that only shows up in one press out of twenty: right-click a sprite, then right-click that *same* entity's row, and both windows are open at once, about one entity, because neither panel's ways-out list (U39.3) contains "the other panel opened it". Every cheap patch for that is a panel reaching into a panel.

**So the anchor is a provider above the docking layout, holding at most one window: `{ owner, scene, entity, at }`.** A panel draws it only when `owner` is its own, and opening from either door overwrites the one slot — which closes the other panel's window without either panel knowing the other exists. `at` is in the owning panel's own pixels and meaningless in any other, which is exactly why `owner` has to be in the state rather than inferred.

**What is shared and what is not, is decided by who can answer the question.** The *placing* rule is shared (one `popoverSpot(panelBox, at)` — next to the click, clamped inside the panel, so the window cannot behave like two different windows depending on the door). The *ways-out* stay with each panel, because they are not the same list: the viewport's anchor is taken away by the camera moving, the Outliner's by the list scrolling, and neither panel can answer the other's question. Both keep the shared half — the entity going, the selection moving off it — in their own render.

This is U9's split one level up, and the sibling of the shared-action hooks (`useDuplicateEntity`, `useDeleteEntities`): there, one *action* with two buttons; here, one *window* with two ways of opening it. **The test that proves it is the one that opens it from both doors and asserts `toHaveCount(1)`** — every other assertion in the file passes with two windows open.

**Gotcha, and it cost a red test:** React's synthetic `stopPropagation` stops the *native* event too. A row that swallowed its own `contextmenu` so the list's background handler would not close the window it just opened also hid the event from a window-level listener — which is how the existing spec asserts the browser's own menu was told no. The fix is to have the container handler **ask where the press landed** (`event.target.closest('[data-entity-id]')`) rather than have the row stop it. Worth reaching for generally: a container that must ignore presses on its children can read the target instead of asking the children to be quiet, and it leaves the event observable. _[earned 2026-08-15]_

### U45: A control used once an afternoon does not hold room that is read all day — put it behind a menu, and give the room it leaves a sentence

The Assets panel carried a permanent make-a-file row under the folder listing: a name field and two buttons, on screen always, used when a level is started. It moved behind two doors — a `+` on the bar, and a right-click on the empty part of the browser — which is U44's shape again (one anchor in the panel, `{ from: 'bar' } | { from: 'browser', at }`, so the two can never both be open). Three things generalise:

1. **The two doors are the two places a hand already goes**, and they are not interchangeable-by-accident: a button on the bar is discoverable, a right-click on the background is what a file browser has trained everyone to try. Building only the second would be a feature nobody finds; only the first is a feature nobody reaches for.
2. **Only the *background* opens it.** A right-click on a row was left to the browser's own menu, because nothing was built for a file yet — a dead right-click that suppresses the machine's menu and offers nothing in its place is worse than the menu. Container asks where the press landed (`closest('[data-asset-path]')`), per U44's gotcha.
3. **The room a moved control leaves is not automatically free**, and this is the trap. The footer's fixed share is load-bearing (UG8: it must not resize when the selection changes, or the first click of a double-click moves the tile the second click is aimed at), so emptying it gains the browser nothing and *looks* broken. The answer is to shrink the reservation to what the remaining control needs and **spend what is left on a sentence naming the gesture that just moved** — the panel's own "where the modifiers are learned" note, which is the only place on screen a right-click could be taught.

**A menu holding a form, not a list, is still a menu**: the same card, the same dismissal, the cursor in the field on open, and — deliberately — no memory of what was typed when it closes. A field that remembered an abandoned name would be a menu pretending to be a panel. _[earned 2026-08-15]_

**Finishing it — the rename/move/delete controls followed, and point 3 turned out to have a better answer than the one it gave.** The press on a row that point 2 deliberately left dead now opens a second card, and the panel below the listing became one unchanging sentence.

4. **When the *last* selection-dependent control leaves a reserved footer, the reservation goes with it — and only then.** UG8 said "reserve a fixed share so a control appearing cannot resize the browser under a double-click", and while any such control remained, that was the whole answer. With all of them in floating cards, nothing down there changes with the selection, so the constraint is satisfied by *removal* rather than by reservation and the browser gets the room. **The lesson is to re-ask an old constraint when the thing that caused it leaves**: point 3's "shrink the reservation and spend the rest on a sentence" was right for a half-finished move and wrong once the move finished, and nothing would have failed to say so.
5. **Which of two menus a press opens is decided by what it landed on, in the one handler.** A file gets the verbs you apply *to* a file; the background gets the verb you apply *in* a folder. Both are the same union of state (`{kind:'bar'} | {kind:'browser',at} | {kind:'file',path,at}`) so two can never be open at once, and the union is preferred to flags-plus-optional-point for `selection.tsx`'s reason: a pair can spell a file menu with no file.
6. **A menu that hands over to another menu needs no arguments, if the press that opened it selected something.** "New level or prefab here" is one `setMenu` to the other card at the same spot: the folder is already right, because right-clicking the file selected it and the make-a-file card has always taken its folder from the selection. Features that look like they need plumbing often need only the ordering.
7. **A control built as a row does not become a card by being put in one.** The rename controls were a name, a folder chooser and two buttons laid out across the foot of a panel; dropped into an absolutely-positioned box, which is shrink-to-fit, they asked for 550 pixels and got them — a menu wider than the panel. It needs a stated width and a stacked layout scoped to the menu. Expect this for any control being moved from a panel edge into a floating card; the layout was written for a width it no longer has. _[extended 2026-08-15]_

### U46: A setting that governs where things land is drawn from the renderer's own report, appears and goes with the switch that governs it, and is not drawn at all rather than drawn wrong

The snap had been a switch and two numbers for two sessions with nothing on screen for it — a board lining up to lines nobody could see. Drawing it is four decisions, and three of them are about *when not to*.

1. **Drawing arithmetic is a different file from placement arithmetic, and only the drawing one reads the switch.** `snap.ts` answers "where does this land" and deliberately does *not* consult `on`, because `Ctrl` inverts the switch and the inverted case has to reach a grid the switch says is off (U42). `grid.ts` answers "is there a grid to draw, and where do its lines fall on this canvas", and checking `on` there is exactly right because nothing inverts a drawing. **The asymmetry is worth a comment in both files**, or the next reader tidies one of them into the other and silently breaks `Ctrl`-with-snapping-off.
2. **The lines come from the renderer's report of where the scene's origin landed, never from a second reading of the camera** — U16 again, with teeth. A grid derived independently agrees with the sprites at 1× and drifts at every other zoom, and the failure mode is a picture that says a tile is aligned when it is a fraction of a cell out. The offset is part of this: a grid drawn through the origin while the snap lands on `16 from 8` is convincing, wrong everywhere, and passes any test that only checks that lines exist.
3. **A grid whose cells are smaller than about six pixels is not drawn at all, and nothing coarser is substituted.** Drawing every fourth line instead would be *true* — a subset of a grid is still that grid — and it would put a spacing on screen that is not the number in the interval field, which is the one thing the drawing exists to show. So it disappears, and both ways back (zoom in, raise the interval) are immediate. The cost is that the editor's own default (`1 from 0`) shows nothing at 1×, which has to be said in the human's page or it reads as the feature being broken.
4. **One tiled pattern, not a line per cell.** A panel-wide grid of six-pixel cells is several hundred elements that React would rebuild on every frame of a pan. An SVG `<pattern>` at `userPatternUnits` with the phase in its `x`/`y` is one element at any density. The price is that **there is nothing to count from the outside**, so what the grid *is* — its interval in level units and its cell in pixels — is published as data attributes on the group instead, and the browser test reads those. A pure function returning `{step, cell, from} | null` keeps the interesting half (does it draw, where do the lines fall) unit-testable without a canvas; the round-trip assertion to write is **take a position the snap can reach, ask the camera where it landed, and check a line is drawn exactly there.** _[earned 2026-08-15]_

### U47: An inspector generated from a description is for the fields the *kernel does not own* — and the panel it replaces is the sentence that said it had no controls

The Inspector had four component types with hand-written controls and one sentence for everything else: *this entity also carries patrol, which this editor has no controls for*. That sentence was the last hole in "the human never reads code" — a component a game invented could be carried, drawn and run, and only authored by typing into the level file. It is closed by the game describing its own components (`text-formats` T22) and the panel drawing whatever the description says.

**The fork worth naming before the mechanics.** The cheap answer is a `patrol` panel written the way `spin` and `screen` are written next door — an afternoon, and it closes the hole for one word while leaving the shape untouched: every noun a game invents still needs a kernel change, which is exactly the ossification the open component map exists to prevent. The expensive answer is the description, and the thing that makes it worth its indirection is that **the kernel never learns the word.** The test that proves you built the second one and not the first is a browser test that *deletes the description file and asserts the fields disappear*. Every other test in such a feature passes for a hand-written panel too.

**Where the line falls, since "generate the inspectors" is the obvious next thought and is wrong.** Texture import settings stayed hand-written and should. The kernel owns them: three fields, the same three in every project, each deserving a sentence about what it does to the pixels. Generation earns its keep only where the *shape is unknown to the code drawing it*. Generating a form whose fields you could have typed out is a description file spent to arrive back where you started with worse prose.

Six things that only show up once it is built:

1. **A description is found by listing a folder, not by being pointed at** — which is the one structural difference from prefabs. A level carrying `patrol` says nothing about where `patrol` is described, so the folder is the index. The *following* is still the shared reference mechanism with an empty witness, and the empty witness is honest rather than a shortcut: an id exists to notice that the file at a path is not the file somebody wrote against, and nothing was written against this file's identity.
2. **Add and Remove are buttons, not a sentinel value.** `spin` deletes itself when its rate is typed to zero, which is right for one number whose zero means "does not turn". A component with three numbers has no such value, and inventing one would be the kernel deciding what a game's component means. The general rule: **a sentinel that deletes is only available to a shape the kernel owns.**
3. **A generated field writes by spreading, where a hand-written one replaces.** The description names the fields it knows; a key it does not name is a key some system reads. Replacing the component object — which is correct for `spin` — silently deletes that key on the first keystroke, and the two lines look identical in review.
4. **A registered format with no case in the inspector's dispatch gets the fall-through body.** The dispatcher ended `if PREFAB … if PROJECT … return <SceneBody>`, so registering a format and stopping there does not produce a blank panel or a crash: it describes a component description as a level, convincingly. **Any `format`-keyed dispatch ending in a default is a place where adding a format is two edits, not one** — worth checking on sight the moment a registry gains an entry.
5. **The panel has to be able to say "the file disagrees with me".** ~~A value of the wrong kind shows the description's default; a default shown as though it were the file's own number is U10's failure exactly.~~ **Amended the same day:** a value of the wrong kind is shown *as the file has it*, read-only, with no control at all — the description's default in a live box was U10's failure in a milder form (a number on screen the file did not hold, one keystroke from being written over the file's word). So the reader returns *what to show*, *whether it is really that*, and *the raw thing held*; the section shows the raw thing and carries a line naming the fields that disagree; and Remove-then-Add is the way back to a value the panel can draw. The same read-only row is what a field of a kind the editor does not know gets. It is also the assertion that stops a well-meaning panel from "fixing" the file it cannot read.

6. **Six kinds, and the two that are references reuse the two pickers — which were lifted the day the second owner arrived.** `TexturePicker` and `SceneMusicPicker` each carried the "fetch the id, write both halves" body, and their own comments warned that a second copy was a second chance to write half a reference; the third owner (a described `asset` field) is what made them one `AssetRefPicker` parameterised by which files to offer. The level picker came out of the project-settings panel for the same reason and by the rule its comment stated: not shared until a second thing pointed at a level (U25). **A picker owns the row, the empty option's sentence is the caller's** — "no sprite", "silent level", "no level chosen" — because what null means is a fact about the owner, not the control. Typed kinds (number, text) merge keystrokes into one undo step; a tick, a pick and a chosen file are one step each.

_[earned 2026-08-15]_

### U48: A picture of a file is keyed on what would change it, read when it is looked at, and kept nowhere the human can see

The Assets panel's icon view drew a folder or a blank-page glyph on every tile, so the one view meant for browsing art identified it by filename. Tiles show the art now. Five decisions, and the first two are the ones that generalise past thumbnails to anything derived from a file.

1. **The identity of a derived thing is every input that would change it, and all of them are already in the folder listing.** A picture's key is `path @ file-mtime @ .meta-mtime` — so re-exporting the art *and* re-slicing the sheet each produce a different picture, and **nothing has to be fetched to find out whether a kept copy is stale**. There is no invalidation code anywhere: a stale copy is unreachable rather than evicted, because nothing can ask for it. The tree already carried both timestamps; the `.meta`'s had to be lifted into the row (`asset-rows.ts`) beside the boolean that says it exists, which cost two lines and removed the whole question.

   The tell that a key is missing an input is a picture that is *right* and *old*: the frame size changes in the Inspector and the tile keeps showing yesterday's crop, which reads as caching being broken rather than as a key being short.

2. **Read on visibility, not on arrival.** One `IntersectionObserver` for the grid (never one per tile), watching the picture boxes, with a rootMargin of a row so a tile is usually ready before it is looked at. A folder of two hundred sprites therefore costs the thirty on screen. Four reads at a time, **newest first**, because a fast scroll asks about a hundred tiles in a second and serving them in order serves the ones the hand has already left.

3. **A derived cache belongs in memory, and the reasons not to put it in the human's folder are not only the marking rules.** `.thumbs` is reserved, and stays empty. A generated binary in somebody's art folder is a file they have to think about — it needs the `generatedBy` marking, it wants a `.gitignore` line, and it makes the editor something that writes into `assets/` unasked. It also buys real problems: stamping each copy with what it was made from, sweeping orphans when art is renamed, deciding what two windows on one project do. What it buys back is a warm start after a reload, and a screenful of small files is read again in well under the second the human is already waiting for. **Keep the decode, throw away the source:** the full-size image lives only long enough for `createImageBitmap` to cut a frame out of it and stand that down to the box, so an entry costs the same whether it came from a 16-pixel sprite or a 4096-pixel tileset, and the bound is a count rather than a guess about anybody's art.

4. **One transition per tile, ever: glyph to picture.** A tile with no picture yet shows the same blank page it always showed — no spinner, no shimmer, because thirty spinners appearing and vanishing down a scroll is the flicker the feature exists to avoid. **A refusal is remembered exactly like a picture**, under the same key, so a file that is not readable art keeps its glyph, says why on its tooltip, and is never read again; fixing the file moves its timestamp, which changes the key, which is the retry. The one thing that does read twice is a *brand-new* file — once as it lands and once when the sidecar writes the `.meta` beside it — which is the key doing exactly what it should and is worth saying in the test rather than working around.

5. **A picture box is height every tile pays for, and the size is chosen against the panel rather than against the art.** 64 pixels looked obviously right and quietly broke a *different* feature: two rows of tiles then filled the Assets panel as it opens, leaving no background to right-click, which is one of the two doors onto making a file (U45). 48 keeps two rows and their background, and is still three whole pixels per pixel of a 16-pixel sprite. **Whole steps only when enlarging** (U17's rule, unforked): filling the box exactly is given up rather than draw some rows of a sprite two pixels tall and some three. The number is one number — the stylesheet takes the box size as a custom property from the module that scales against it — because two would drift and the symptom would be a picture hanging out of its tile.

Two smaller things. The `.meta` is read for the slicing but **not offered to the document store**, unlike the selected file's (`useAssetMeta.ts`): adopting thirty documents because thirty tiles scrolled past would fill the store with things no inspector will ever open. And what the tile *drew* — the frame it cut and the image it cut it from, `16x16` out of `64x16` — is published as data attributes, which is how "the strip is not a smear" is asserted without comparing pixels (U16's habit, one panel over). Every other assertion in such a feature passes for a tile showing the whole sheet.

_[earned 2026-08-15]_

## Gotchas

### UG19: An SVG `<pattern>` clips its own tile, so a stroke on the tile boundary comes out half width — and a React `useId` cannot go straight into `url(#…)`

Two small things that both look like styling problems and are not.

**The tile is a clip.** Drawing the grid line at `x = 0` or `x = cell` puts a 1-pixel stroke centred on the tile's edge, and the pattern clips everything outside the tile — so half the stroke is thrown away and the line renders fainter than the same line drawn anywhere else, at some cell sizes and not others. Draw the lines a half-pixel *inside* the tile (`M 0.5 0 V cell`) so the whole stroke is within it. The visible symptom is a grid that appears to fade at certain zooms, which reads as an opacity bug.

**A React id is not URL-reference-safe.** `useId()` returns something like `«r1»` in React 19, and `fill="url(#«r1»)"` does not resolve. A per-instance id is still the right call over a module constant — two overlays with the same pattern id do not fail loudly, the second one quietly paints the first one's spacing — so sanitize it (`useId().replaceAll(/[^\w-]/g, '')`) rather than reaching for the constant. _[earned 2026-08-15]_

### UG18: A menu that handles `Escape` on its own subtree must take the focus when it opens, or the key lands on whatever opened it

Every menu in this editor stops `Escape` on its own element rather than on the window, deliberately: the viewport owns that key for calling off a grab (U33), so whichever thing is open should be the one that hears it. The unstated half is that **the key only reaches the menu if something inside it has the focus.**

The make-a-file card and the entity window both got away with it by accident — each autofocuses its first field because that is the obvious thing to want. The file menu did not, and `Escape` did nothing at all: the right-click left the focus on the row it pressed, so the keydown went to the row and the menu never saw it. The failure is silent and reads as "Esc is broken here", not as "focus is somewhere else".

**Fix/policy: a floating menu takes the focus when it opens** — the field a hand is about to type in, or the card itself with `tabIndex={-1}` when it has no field. The test that catches it is one line (`await expect(firstField).toBeFocused()`), and it is worth writing next to the Escape test rather than instead of it: the two fail for different reasons. Sibling of the entity window's rule about handing focus *back* on close (U39). _[earned 2026-08-15]_

### UG17: A percentage `max-height` on a floating card resolves to nothing, and a content-box one is short by its own padding

A card that can outgrow its panel needs a ceiling, or the panel's hidden overflow clips it with no scrollbar to say so — buttons simply missing from the bottom of a menu. Two things went wrong in a row on the way to getting one:

1. **`max-height: calc(100% - 16px)` computed to none.** A percentage `max-height` resolves against a containing block with a **definite** height, and a panel sized by a flex parent has a height that *measures* 262 and is not definite. The computed style still reports the `calc()` unresolved, which reads like it is working. The fix is to state it in **pixels**, from the panel rect the placement function has already measured — the number is right there, and passing it down as an inline style removes the question. (`spotIn` returns it beside `top`/`bottom`, so the anchored edge and the room from it are decided together.)
2. **Then the card was still ten pixels too tall**, because the stylesheet is content-box throughout: `max-height: 246px` bounds the *content*, and the card's padding and border sit outside it. A ceiling meant as "the room in the panel" has to be a border-box ceiling. Stated on the card rather than globally, since changing `box-sizing` for the whole editor would move every control in it.

The tell for both is a `max-height` that is present in the computed style while the element measures larger than it. _[earned 2026-08-15]_

### UG16: "Scale" is already taken in a 2D editor — the camera's zoom is a scale, and so is an entity's size

Adding the `S` gesture wanted `data-testid="scene-scale"` for its gizmo. That id already existed, on the zoom readout in the viewport's caption: the camera's scale. Two nodes, one test id, and Playwright's strict mode failed with "resolved to 2 elements" in the two tests that looked for the gizmo — a legible failure, but only because a *locator* hit it. The same collision in a data attribute read by `getAttribute` would have returned the wrong number silently.

The editor now has three unrelated meanings of the word within one panel: `data-scene-scale` (the camera's), `data-scene-scale-x/-y` (the gesture's factor), and each entity's own `scaleX`. **Fix/policy: name a gesture's marks after the *gesture*, not after the quantity** — `scene-scaling`, `data-scene-scaling` — which is unambiguous precisely because it names something that is happening rather than a number that three things have. Worth checking on sight whenever a new gesture shares a word with the camera: "scale", "position", "origin" and "bounds" all already mean something in a viewport. _[earned 2026-08-15]_

### UG15: A listener effect that reads `ref.current` arms against null when its element renders later — and every other code path heals it, so only the first gesture of a session is unguarded

The side-button guard (UG14) was attached in an effect whose first line was `const element = surface.current; if (element === null) return`. The Assets panel shows "Reading the project folder…" before the browsing area exists, so on the effect's first run the ref was null and the guard armed against nothing. The effect's deps — the view, the trail's callbacks — all change on ordinary use, and any of them re-runs the effect and arms it for real.

**That healing is what made it invisible.** Every browser test switched views or walked folders before pressing the side buttons, so the tests' own setup armed the guard they were testing. The hole only opened when the icon view became the *default*: a session's very first side-button press, before anything else was touched, reached an unguarded surface — and Chromium navigated the editor to `about:blank`. UG14's exact blank-window symptom, arriving through a different hole, found by the one test that presses back before doing anything else.

**Fix/policy:** an element a listener effect needs is passed as **state** (callback ref into `useState`), so the element's appearance is itself a dependency and the effect re-runs when it mounts. A plain ref remains right only for lazy reads inside event handlers (the split handle reads its ref mid-drag and never in an effect). Worth checking on sight: any effect beginning `const el = ref.current; if (el === null) return` where the element can render after the first commit — the tell is a guard or subscription that works everywhere except a freshly loaded page. Sibling of UG5: both are state read at effect time answering for a render that has already moved on. _[earned 2026-08-14, React 19.2.8]_

### UG14: Chromium navigates on the *release* of a mouse's back button, so cancelling the press looks like it worked

The mouse's fourth and fifth buttons are a file browser's back and forward, and Chrome maps them to the browser's own history. A page that wants them has to cancel the browser's gesture, and the obvious place is `mousedown` — which is where `preventDefault` visibly works: the event reports `defaultPrevented`, the folder changes, and nothing appears to be wrong.

**Chromium acts on `mouseup`.** So the press cancels, the release navigates, and the single-page app is replaced by whatever the tab held before it. In a window opened straight at the editor that is `about:blank`, so the first symptom is not "the wrong folder" — it is a blank white window and a lost session, arriving a moment after a gesture that appeared to work. Firefox acts on `auxclick` instead, so all three need cancelling.

**Fix:** cancel `mousedown`, `mouseup` *and* `auxclick` for `button === 3 || button === 4`, and cancel them whether or not the gesture does anything — the editor never wants the page navigated, and a rule with an exception is a rule that gets found by the exception.

Worth knowing for the test as much as the code: **Playwright's `mouse` API has three buttons and no more**, so the only way to press this one is `Input.dispatchMouseEvent` over a CDP session with `button: 'back'` and the held-buttons mask (8 for back, 16 for forward). That is the real button as the browser reports it — a synthetic `MouseEvent` from inside the page would prove the handler is wired up and say nothing about whether the browser had already taken the press, which is the entire failure. _[earned 2026-08-14, Playwright 1.62.1's Chromium]_

### UG13: A control above a browsing area resizes it, and the first half of a double-click then moves the target of the second

The Assets panel's make-a-file and rename rows sat above the tree, where they had been harmless for two sessions: their height changes with the selection — the rename row does not exist until something is selected, and it says a different number of things about a folder than about a file. The moment the panel grew a view whose gesture is a **double-click**, that became a bug with no error in it. The first press selects the folder, the rename row comes into existence, every tile below it moves by that row's height, and the second press lands on whatever slid into that spot. No `dblclick` event is ever dispatched — the browser only fires one when both presses hit the same element — so the folder never opens and nothing anywhere reports a thing.

Moving the controls *below* the browser does not fix it. In a short panel a footer that grows takes its room from the bottom of the browser, and the boundary sweeps up across the pointer instead of down: the second press landed in the name field of the make-a-file row.

**Fix:** the controls hold a fixed share of the panel and scroll inside it — `height: min(40%, 150px)` — so nothing they contain can change the size of the browser above them. A share *and* a ceiling, because either alone is wrong: a proportion wastes a big panel and a fixed height swallows a small one, and a bare pixel count is a guess about a font (UG10).

**The general form, and it is UG8 in a panel with no canvas in it: what a gesture is aimed at must not be moved by the first half of that gesture.** Worth checking on sight wherever a multi-press gesture shares a box with anything that changes size on selection — and the tell is unmistakable once seen, because the failure is *silence*: a double-click that does nothing at all looks exactly like a handler that was never wired up, which is where a session will go looking first. Confirming it took a document-level `dblclick` listener logging its target; the answer was the target being a text input three inches away. _[earned 2026-08-14]_

**The reservation is gone as of 2026-08-15, and the constraint is not.** Once every one of those controls moved into a floating right-click menu (U45.4), nothing beside the browser changed size with the selection any more, so the fix became *removal* rather than reservation and the panel gave the room back. **A floating card satisfies this rule by construction — it is out of the flow, so it moves nothing.** Which is the general note to carry: when a control that changes size must live beside a gesture surface, floating it is a stronger answer than reserving room for it, and the reserved room should be re-examined the moment the last such control leaves.

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

**Half fixed 2026-08-14: the panel menu exists** — the Windows menu on the status strip (U38) reopens any closed panel, so the dead end is gone. Layout persistence is still the other half: the *arrangement* resets on reload, and that remains the feature saved layouts will be. _[amended 2026-08-14]_

## Contracts

Contracts are referenced as file paths, never paraphrased as prose. Read the file; don't trust a summary of it.

- `kernel-2d/editor/shell/panels.tsx` — every panel the editor has and the layout it opens in. Adding a panel happens here and nowhere else (U1), and the count of live renderers is bounded by what is in here (U18). A panel gains a real body by getting a `render`; without one it shows its own description, which is what keeps unbuilt panels honest instead of blank. All five have bodies now. Also the wrapper that puts all but the Viewport out of reach while a level runs (U26).
- `kernel-2d/editor/panels/AssetsPanel.tsx` — the folder mirror, the worked example of a panel with a body, U22 (the control that puts a file in a human's project) and U29 (the one that renames, moves and deletes one). Also U34's frame — a bar, two halves either side of it — and U45, where every verb the panel has left the footer for a right-click menu and the footer became one sentence.
- `kernel-2d/editor/shell/asset-browsing.tsx` — U34's state and U36's: which view, the trail of folders and where along it you are, how the split is divided, which folders are open, and the rename hook U30 requires of all of them.
- `kernel-2d/editor/shell/useFolderHistoryButtons.ts` — the mouse's side buttons, UG14's three cancellations, and UG15's rule that the surface arrives as state rather than a ref.
- `kernel-2d/editor/panels/SplitHandle.tsx` — U36's divider: a fraction rather than a width, a hairline inside a strip, and the double-click that puts it back.
- `kernel-2d/editor/panels/AssetGrid.tsx` — the icon view. One folder at a time, the same rows the tree gets, and U48's watching half: the one observer that asks for a picture when a tile comes into view, and the fixed box that keeps the grid from reflowing under a double-click.
- `kernel-2d/editor/shell/thumbnail.ts` — U48's arithmetic, with no React and no browser in it: what identifies a picture, which frame is cut, how big it is kept, and the whole-step rule for how big it is drawn. Also why the box is the size it is, which is a fact about the panel rather than about the art.
- `kernel-2d/editor/shell/thumbnails.tsx` — U48's cache: every picture the window has drawn, the queue that bounds how many are read at once, the eviction rule that may not touch what is on screen, and the paragraph on why none of it goes to `.thumbs`.
- `kernel-2d/editor/panels/AssetBar.tsx` — the breadcrumb, the `+` that makes a file (U45), and the cog behind which the three views live — including why the breadcrumb is not on screen when the tree is, and why an action and a setting do not share a menu.
- `kernel-2d/editor/panels/NewDocument.tsx` — U22 and U45: the make-a-file card, rendered by whichever of the three doors opened it.
- `kernel-2d/editor/shell/scale.ts` — U33's third gesture: the factor pair, the ratio, and what "local" means for a group.
- `kernel-2d/editor/shell/useMenuDismiss.ts` — what every menu in the editor does about Escape and a press elsewhere, lifted at the third caller.
- `kernel-2d/editor/shell/floating.ts` — where a small floating card sits when it is opened at a spot in a panel: below the press, flipped above it, pinned to an edge last (U39.8), with the room it has stated in pixels (UG17). Shared by the entity window and both of the Assets panel's menus (U45).
- `kernel-2d/editor/shell/useFileMoves.ts` — the whole of a rename as a gesture: flush, plan, refuse, move, rewrite, report, and U30's remapping of what the human was looking at. The comment on why none of it goes through `edit` is the load-bearing one.
- `kernel-2d/editor/shell/references.ts` — which documents point at a file, and the same documents with its new path written in. Only `path` moves; `id` never does.
- `kernel-2d/editor/store/file-disk.ts` — the editor's two file operations, beside `document-disk.ts` and `meta-disk.ts`.
- `kernel-2d/editor/store/service.ts` — `askService`: how every one of those writers asks the editor service — send, throw the service's own sentence on a refusal, parse the answer against its schema before believing it. A fourth writer calls it rather than re-writing the four lines (`editor-kernel` G3, run four).
- `kernel-2d/editor/store/open-documents.ts` — the one document store per window and the hooks panels read it through. The API itself is `editor-kernel`'s (D7); this is where the UI meets it, and where editing is suspended while a level runs (U26).
- `kernel-2d/editor/panels/InspectorPanel.tsx` — the worked example of U10, U11 and U12: every state it can be in has a sentence, and the editable one is reached only from the store. Three document formats have a body of their own now, each reached by what the document says it is.
- `kernel-2d/editor/panels/ProjectInspector.tsx` — U28: the third inspector body, the startup-level picker that reads a file before it writes a path, and the sentence that says what the current choice resolves to.
- `kernel-2d/editor/shell/zoom.ts` — what is left of the zoom controls after the ladder moved into the shipping layer (`editor-kernel` D20): stepping, and the wording.
- `kernel-2d/editor/panels/TextureSettings.tsx` — the first editable controls, hand-written, and the worked example of U14. Every one of them goes through the transaction API and none of them knows undo exists.
- `kernel-2d/editor/shell/useUndoShortcuts.ts` — U13, and the modifier-only half of the editor's keyboard.
- `kernel-2d/editor/panels/ViewportPanel.tsx` — the scene, and the worked example of U10 at its widest: no scene open, opening, gone, unreadable, empty, a texture it cannot draw, everything off screen, and the selected entity off screen are eight different sentences. The last two arrived with the camera, and they *replace* the count rather than joining it — see UG8 for why that is layout as well as prose. Also the right-click window's anchor, clamping and ways out (U39).
- `kernel-2d/editor/panels/TexturePanel.tsx` — U15 and the single-texture preview, moved out of the Viewport when the Viewport became the scene.
- `kernel-2d/editor/panels/TextureOverlay.tsx` — U16. Frame guides, the strip no frame reaches, the pivot marker, and the caption that says in words what the shading says in pixels.
- `kernel-2d/editor/panels/SceneOverlay.tsx` — U16 again: the grid the snap lands things on (U46), the selected entity's outline, the crosshair on its position, the line a locked grab runs along (U33), wherever the camera has put the corner the scene's y counts up from, and which entities are actually on the canvas.
- `kernel-2d/editor/panels/OutlinerPanel.tsx` — the Outliner (named Hierarchy until 2026-08-14; older entries here say Hierarchy): what is in the open scene, in draw order, and the actions that change it — every one a transaction and none of them aware undo exists. Reordering has two doors, the arrows and dragging a row (U37); a right-click on a row is the second door on the right-click window (U44).
- `kernel-2d/editor/panels/EntityInspector.tsx` — the second inspector, and where a D5 reference is written.
- `kernel-2d/editor/panels/EntityPopover.tsx` — U39: the right-click window itself — rename, position, Frame, Duplicate, Delete — its shared merge keys, why its propagation stops are native listeners, and why none of its buttons closes it. Rendered by two panels; owned by neither.
- `kernel-2d/editor/shell/entity-popover.tsx` — U44: the one anchor that window hangs off, its `owner`, and the placing rule both doors share.
- `kernel-2d/editor/panels/PrefabInspector.tsx` — the inspector body a prefab document gets: what a thing is, the button that puts one in a level, and the note on why placing is an edit to the level rather than to the prefab.
- `kernel-2d/editor/panels/NumberField.tsx` — U14, shared the moment a second inspector wanted the same behaviour.
- `kernel-2d/editor/shell/viewport-context.tsx` — U9's third case: the texture renderer, above the layout, and the zoom state of U17.
- `kernel-2d/editor/shell/scene-view-context.tsx` — U18 and U19: the scene renderer, why two is a bounded number rather than a habit, one camera per scene for the life of the window, and the three conditions a scene satisfies before it is framed. Also `useDrawScene`: the request key that distinguishes two derivations of one level, and the answer to U27.
- `kernel-2d/editor/shell/useSceneGestures.ts` — U20, U21, U31, U33 and U39: left-press to pick, place or stamp, right-click to ask (or to leave a mode), middle-drag, space-drag, wheel-to-zoom, the framing keys, `G`, `R` and `S` with their axis keys, `Shift-D`, Esc, the order they take priority in, and the unconditional suppression of the browser's context menu.
- `kernel-2d/editor/shell/useDuplicateEntity.ts` — U33's fifth point: one answer to what a copy is, called by the Outliner's button and by the viewport's key.
- `kernel-2d/editor/shell/drawn-entities.ts` — every question asked about the picture the renderer drew: what an entity covers, what is on the canvas, and what is under the pointer. One set of rectangles, so a click cannot disagree with an outline.
- `kernel-2d/editor/shell/open-scene.tsx` — which scene is open and the document behind it; one fetch that both decides a `.json` is a scene and puts it in the store.
- `kernel-2d/editor/shell/scene-assets.tsx` — U9 for a set whose membership changes: every texture a scene refers to, resolved once per window.
- `kernel-2d/editor/shell/layout-context.tsx` — the handle on the docking layout: bringing a tab forward for a selection, and the Windows menu's focus-or-spawn (`summon`, U38), with the note on why the two stay different verbs.
- `kernel-2d/editor/shell/useSelectionFocus.ts` — U18's third practicality: which tab comes forward for what was just clicked.
- `kernel-2d/editor/shell/asset-meta-context.tsx` — U9's second case, and why a fetch with a side effect is never a panel's to own.
- `kernel-2d/editor/shell/asset-kinds.ts` — what a file is (U11) and how to find it in the tree. Shared the moment a second panel needed the same answer.
- `kernel-2d/editor/shell/asset-rows.ts` — which rows a folder has, and why a sidecar folds into the row of the file it annotates (`editor-kernel` D4). Shared because the Inspector counts a folder the same way the panel lists it. Carries *when* those folded-in settings were written as well as whether they exist, which is what lets anything derived from them know it is stale without asking (U48).
- `kernel-2d/editor/shell/selection.tsx` — U8. What is selected, which scene is open, and nothing else.
- `kernel-2d/editor/shell/scene-prefabs.tsx` — U23's source: the resolved level every panel draws and describes, and the loud note saying it is never the thing that gets written.
- `kernel-2d/editor/shell/useReferences.ts` — following a reference by path and modification time, once: the read-token ordering, the ask-once rule, and why "not ready" is not a problem. Shared by the textures a level draws and the prefabs it places from.
- `kernel-2d/editor/panels/fields.tsx` — the four pieces every inspector body is built from, and the note on which lookalike deliberately did not move in.
- `kernel-2d/editor/shell/entity-names.ts` — one free-name helper, and the two things that actually differ between its callers.
- `kernel-2d/editor/shell/usePlacePrefab.ts` — U24 as built: one gesture, reachable from the prefab and from what it just placed. Two doors rather than one optional argument, and the note saying why a button's `onClick` is what forces that. The prefab comes from the store here; what an instance *is* does not (U35).
- `kernel-2d/editor/shell/place-into-scene.ts` — the two recipes for something new in a level, shared by every caller that puts one there: an instance of a prefab, and an entity that draws a texture. Plain functions, no hooks, no undo code.
- `kernel-2d/editor/shell/useAssetDrag.ts` — U35's source half: what a file being carried out of the Assets panel puts on the drag, and why the path is not on it.
- `kernel-2d/editor/shell/useSceneDropTarget.ts` — U35's target half: the three events, the enter/leave count, and the inversion through the camera the renderer drew with.
- `kernel-2d/editor/shell/useDropIntoScene.ts` — what a dropped file turns out to be, read rather than guessed, and the sentence for each thing it cannot be.
- `kernel-2d/editor/shell/grid.ts` — U46: whether there is a grid to draw at all, how big its cells are on screen, and where its lines fall. The one place the Snap switch means "draw", and the paragraph on why the minimum cell size is not answered with a coarser grid.
- `kernel-2d/editor/shell/snap.ts` — U31's arithmetic, the paragraph on why a grid needs an offset as well as a step, and the three intervals one switch governs (grid, angle, factor). No React in it, so the properties are unit-testable.
- `kernel-2d/editor/shell/placing.tsx` — U31's state: what a press lands on and what it puts down, above the layout, in neither the document nor any file.
- `kernel-2d/editor/panels/PlaceByClicking.tsx` — the switch, shared the moment the second inspector wanted it, and the note on why turning it on places nothing.
- `kernel-2d/editor/panels/TexturePicker.tsx` — one picker, two owners, and the D5 round trip that fetches a texture's id. Lifted out the moment a prefab needed the same control as an entity.
- `kernel-2d/editor/panels/SceneMusicPicker.tsx` — U40: which sound a level plays, written as a D5 reference on the scene itself, and the note on why nothing plays while editing.
- `kernel-2d/editor/shell/component-types.tsx` — U47: this game's own component vocabulary, read once per window from a folder listing rather than from a reference, and what happens when two files claim one type.
- `kernel-2d/editor/panels/ComponentFields.tsx` — U47: the generated fields themselves, one control per kind, the four states an entity can be in about a described component, and the three rules that differ from the hand-written controls beside them (buttons over sentinels, spread over replace, show-and-leave-alone over default-in-a-box).
- `kernel-2d/editor/panels/AssetRefPicker.tsx` and `LevelPicker.tsx` — U47.6: the one place a `{id, path}` reference is written and the one place a level is picked, each owning its row and none of the sentence about what "Nothing" means.
- `kernel-2d/editor/shell/project-context.tsx` — U9. One folder read and one change stream per window.
- `kernel-2d/editor/shell/useAssetMeta.ts` — one file's import settings, re-asked when the folder changes as well as when the selection does.
- `kernel-2d/editor/shell/useProjectTree.ts` — the folder, kept current: the re-read of U5, the settle of U6, and the stale state of U7.
- `kernel-2d/editor/shell/App.tsx` — the shell: status strip above, docking layout below, providers around both.
- `kernel-2d/editor/shell/useSidecarStatus.ts` — how the editor learns which project it is connected to, and what it does when the answer stops coming (U3).
- `kernel-2d/editor/shell/StatusStrip.tsx` — the connection line, including the `data-testid` hooks the browser suite reads, and the Windows menu (U38).
- `kernel-2d/editor/shell/shell.css` — the frame around the docking layout, and **every colour the editor has**: the palette tokens, what the accent is allowed to mean, and the block that repaints dockview's theme in those same tokens (U32).
- `kernel-2d/vite.config.ts` — the editor's root folder, the `/api` proxy to the sidecar (U2), and the loopback binding (UG3).
- `kernel-2d/scripts/editor-server.ts` — the editor window's host, port, and open-a-browser knobs, with their environment variable names.
- `kernel-2d/tsconfig.editor.json` and `kernel-2d/tsconfig.base.json` — the browser half of U4.
- `kernel-2d/editor/shell/play-mode.tsx` — play as window state: what it is, what pressing Play does before it reads the file, and why it never touches one.

**Not yet written** — until a path appears here, the contract does not exist and must not be assumed:

- Inspector auto-generation from Zod schemas. The generated fields that exist (U47) are driven by a *game's* description of its own components, not by a schema — and every inspector body for a format the kernel owns is still hand-written, deliberately, for the reason in U47: generation earns its indirection only where the shape is unknown to the code drawing it.
- **Draggable** handles of any kind — corner boxes for size, a ring for angle, a pivot that can be moved, a frame grid. Position, angle and size are all reachable in the picture, but only as modal key gestures (U33's `G`, `R`, `S`), and every one of them works about the selection's own middle. Also not written: a marquee, and cycling through overlapping sprites with repeated clicks.
- Rulers down the edges of the picture, and a grid while snapping is *off*. The grid itself is drawn now (U46), but it appears and goes with the Snap switch, so laying a level out by eye is still done against nothing. Rulers have not been asked for at all.
- Painting by dragging: a stroke that stamps one copy per cell crossed. Clicking is quick enough, and a stroke is the first step toward a brush, a rectangle tool and an eraser — none of which the kernel has been asked for and all of which are what `genre-spinup` S1 sends at a game folder.
- Nesting in the Outliner. The list is flat; a row drags to *reorder* (U37), never to become a child of another.
- Selecting more than one *file*. Entities select in groups and move, turn, scale and delete as one; the Assets panel still holds exactly one path, so its rename, move and delete verbs are one file at a time.
- Dragging a file inside the Assets tree or grid to move it. Moving is a name field, a folder chooser and a button (U29); drag-and-drop over a tree is a second gesture surface for an operation that already has one. A file drags *into the level* (U35) and nowhere else.
- Any drop target for a *file* but the picture. Not the Outliner (its rows drop into it, but never an asset — the two drags carry different marker types, U37), not an entity in it, and not an entity in the viewport — dropping a texture *onto* a sprite to re-point it is a different operation (changing a reference) wearing the same gesture as creating one, and the Inspector's picker already owns it.
- Accepting a file dragged in from outside the editor. The marker type on the drag is what an OS file drag does not have, so it is refused by construction rather than by a check; copying a file into the project folder is the sidecar's business and nobody has asked for it.
- A picture of anything that is not a picture. Tiles show the art in an image file (U48) and nothing else does: no waveform for a sound, no sample for a font, no rendered preview of a level or prefab. Each is a different feature with different questions in it, and the last one would need a renderer per tile. Also not written: any frame of a sheet but the first, a moving GIF, a choice of tile size, and a thumbnail anywhere other than these tiles — not the tree's rows, not the texture picker, not the drag ghost.
- Sorting, searching or filtering the icon view, and a tile size. The folder's own order (`editor-kernel` D4's rows, folders first) is the only order, and the tiles are one size.
- Back and forward *buttons* in the bar. The trail exists and the mouse's side buttons walk it (U36), the breadcrumb goes back up, and nothing on screen goes forward — so a hand without a five-button mouse has no forward at all. Two arrows in the bar is the obvious answer and has not been asked for.
- Remembering the Assets view or the folder across a reload. It is window state on purpose (U34); persisting it is the same feature as saved layouts (UG4) and belongs with it.
- Deleting a folder. Renaming and moving take folders; deleting is one file at a time (`editor-kernel` D22).
- A panel menu or anything that lists the viewport's shortcuts. `Home` and `F` are named in the caption's own sentences and on two buttons; `G`, `X`, `Y` and `Esc` are named by the caption while a grab is running; `Shift-D` is in the Duplicate button's tooltip. Nowhere else, and there is no list of them in the editor.
- Two scenes open at once. `openScene` is one path.
- Saved layouts (UG4's second half — the panel menu is U38, and the arrangement still resets on reload).
- A keyboard shortcut registry. There are two window-level handlers — the undo one (U13) and the viewport's (U20, U33) — and they do not overlap: the first is modifier-only, the second is bare keys guarded by UG7's typing check. Two do not need a registry; a third that wanted a bare letter would be the moment to think again.
- Anything that shows the undo stack — a history panel, an Edit menu, an "Undo <label>" caption. The labels exist and `peekUndo`/`peekRedo` expose them; nothing reads them yet.
- Any control over a level while it is running: a camera that can be moved, a pause, a step, a frame counter, or editing that the running level picks up. Play inherits the editing camera and freezes it, and the rest of the editor goes read-only until Stop.
