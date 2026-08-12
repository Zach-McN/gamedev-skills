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

### U9: Anything with a live subscription is read once per window and shared, not per panel

The Assets panel and the Inspector both need the project folder. The hook that fetches it and holds a change stream open is called once, in a provider above the layout, and both panels read from that. **Reason:** two callers means two streams, two fetches, and — the part that actually bites — two copies refreshed on separate timers, so one panel can be a beat behind its neighbour with nothing on screen saying so. Same reasoning as U5 one level up: the failure is not the wasted work, it is the two answers. **Providers go above the docking layout**, because dockview mounts and unmounts panel bodies as tabs move, and state held inside a panel is lost the first time the human drags it. _[earned 2026-08-11]_

**Extended twice, and the second reason is stronger than the first.** The selected file's `.meta` moved up here the moment the Viewport wanted it as well as the Inspector. The U9 reason applies as stated — but there is a second one that only shows up when the fetch has a side effect. Asking for the settings is also what *puts them in the document store*, so with the fetch owned by the Inspector, closing that tab would leave the Viewport drawing a texture whose settings never arrived: a panel silently depending on another panel being open, with no error and nothing on screen to explain it. **A fetch that populates shared state is not a panel's to own, whatever the caller count.** Worth checking on sight: if a hook writes somewhere other than its own component, it belongs above the layout.

**A live renderer belongs up here too, and for a harder reason.** A Phaser game owned by the Viewport panel is destroyed and rebuilt the first time somebody drags that tab — the same teardown a per-selection design would have caused, arriving through a different door. Rebuilding means a fresh WebGL context, which browsers hand out in limited numbers (`phaser4-runtime` P2). The canvas is created detached, the panel adopts it on mount and returns it on unmount, and the game never notices. _[earned 2026-08-11]_

### U15: A live canvas is adopted by the panel that hosts it, not created by it

The renderer's canvas is a plain DOM element made outside React and held by the provider. The panel renders an empty host and, in an effect, appends the canvas on mount and removes it on unmount. Sizing is a `ResizeObserver` on the host, reported up to the provider, which tells the renderer.

**Reason:** this is what makes U9's third case work in practice. The canvas cannot be a React child, because moving it between parents on every tab drag would mean unmounting and remounting the element itself. Two details that are easy to get wrong: the game must be given `parent: null` or it appends its own canvas to the document body (`phaser4-runtime` G5); and everything the panel sends *down* (its size) should be separate from everything it sends *across* (what to draw), because a panel being dragged wider is not a reason to fetch anything again. _[earned 2026-08-11]_

### U16: What a panel draws over a canvas is DOM, and its numbers come from the renderer

Frame guides and the pivot marker are an SVG layer above the canvas, absolutely positioned and `pointer-events: none`. Every rectangle in it is one the renderer reported having cut, at the placement the renderer reported having drawn at; the overlay computes no geometry of its own.

**Reason:** two payoffs and one guard. A one-pixel line stays one pixel at 24× where a renderer-drawn line becomes a 24-pixel bar unless it is drawn in screen space, which means screen-space arithmetic inside a game renderer. Text stays selectable and legible. And it is assertable — a frame count, a marker position and a caption are ordinary locators, which is how a canvas feature gets tested without comparing pixels (`editor-verification` V17). The guard is the reason the numbers come from the renderer rather than from the same inputs: an overlay that re-derives the placement is a second derivation that agrees until it doesn't. _[earned 2026-08-11]_

### U17: Zoom for pixel art is whole steps only, fitting by default

The scale ladder is `1/16 … 1/2, 1, 2, 3, 4, 6, 8 … 32` — always a whole number of screen pixels per image pixel or the reverse. Fitting picks the largest step that fits the panel; the step buttons turn fitting off; a Fit button turns it back on, and so does selecting a different file.

**Reason:** at 3.4× some rows of a sprite are three pixels tall and some are four, which reads as *badly drawn art* rather than as a badly chosen zoom — so the human goes looking for the fault in their own work. Filling the panel exactly is not worth that. Two details: selecting a different file returns to fitting, because a zoom chosen for a 16px sprite is not a choice anybody made about a 4096px tileset; and fitting cannot be decided until the image's size is known, so the first draw lands at whatever scale was current and a follow-up settles it. That settles rather than oscillating for the same reason `editor-kernel` G10 does — after one pass the drawn scale *is* the wanted scale, so the condition is false and nothing runs again. _[earned 2026-08-11]_

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

## Gotchas

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

- `kernel-2d/editor/shell/panels.tsx` — every panel the editor has and the layout it opens in. Adding a panel happens here and nowhere else (U1). A panel gains a real body by getting a `render`; without one it shows its own description, which is what keeps unbuilt panels honest instead of blank.
- `kernel-2d/editor/panels/AssetsPanel.tsx` — the folder mirror, and the worked example of a panel with a body.
- `kernel-2d/editor/panels/InspectorPanel.tsx` — the worked example of U10, U11 and U12: every state it can be in has a sentence, and the editable one is reached only from the store.
- `kernel-2d/editor/panels/TextureSettings.tsx` — the first editable controls, hand-written, and the worked example of U14. Every one of them goes through the transaction API and none of them knows undo exists.
- `kernel-2d/editor/shell/useUndoShortcuts.ts` — U13, and the only keyboard handler in the editor.
- `kernel-2d/editor/panels/ViewportPanel.tsx` — U15 and the worked example of U10 generalised: the canvas's host, and every sentence that stands in for a picture.
- `kernel-2d/editor/panels/ViewportOverlay.tsx` — U16. Frame guides, the strip no frame reaches, the pivot marker, and the caption that says in words what the shading says in pixels.
- `kernel-2d/editor/shell/viewport-context.tsx` — U9's third case: one renderer per window, above the layout, and the zoom state of U17.
- `kernel-2d/editor/shell/asset-meta-context.tsx` — U9's second case, and why a fetch with a side effect is never a panel's to own.
- `kernel-2d/editor/shell/zoom.ts` — the scale ladder of U17.
- `kernel-2d/editor/shell/asset-kinds.ts` — what a file is (U11) and how to find it in the tree. Shared the moment a second panel needed the same answer.
- `kernel-2d/editor/shell/asset-rows.ts` — which rows a folder has, and why a sidecar folds into the row of the file it annotates (`editor-kernel` D4). Shared because the Inspector counts a folder the same way the panel lists it.
- `kernel-2d/editor/shell/selection.tsx` — U8. What is selected, and nothing else.
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

- Inspector auto-generation from Zod schemas. The three texture controls are hand-written on purpose: the useful generic version is written once several inspectors exist to generalise from.
- The Hierarchy panel. Still a placeholder; it lands with the scene format.
- Anything editable in the viewport — gizmos, dragging a pivot, dragging the frame grid. The Inspector's controls are the only way to change these values, deliberately.
- The panel menu and saved layouts (UG4).
- A keyboard shortcut registry. There is exactly one handler (U13), and one does not need a registry.
- Anything that shows the undo stack — a history panel, an Edit menu, an "Undo <label>" caption. The labels exist and `peekUndo`/`peekRedo` expose them; nothing reads them yet.
