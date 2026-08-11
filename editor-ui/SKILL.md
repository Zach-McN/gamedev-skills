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

## Gotchas

### UG1: Dockview's layout is computed inside `requestAnimationFrame`, so a non-compositing surface freezes it at 100×100

Dockview sizes itself from a `ResizeObserver` whose callback is deferred through rAF. In any browser surface that is not painting — a hidden preview pane, a background tab, a devtools-detached window — rAF never fires, so the grid keeps its initial 100×100 size, every panel is 100px wide, and every sash reports `dv-disabled`. It looks exactly like a broken layout or a missing CSS import. **Fix/policy:** before debugging dockview sizing, confirm the surface is actually compositing; verify in a real window or in headless Chromium (which does paint). Nothing about the CSS or the container needs changing. _[earned 2026-08-11, dockview 8.0.0]_

### UG2: dockview-react 8 logs a console error about `dockview-enterprise` on every load

`DockviewReact` unconditionally passes a `createContextMenuItemComponent` framework option, and dockview-core logs an error when the matching enterprise module is not registered. It appears once per page load, is not caused by anything the consumer does, and cannot be switched off through props. **Fix/policy:** ignore it, do not install `dockview-enterprise` to silence it, and do not write a "no console errors" assertion into the browser suite — that assertion would be permanently red for an upstream cosmetic. _[earned 2026-08-11, dockview-react 8.0.0]_

### UG3: Vite binds to `localhost`, which is IPv6-first on Windows

The dev server's default host resolves to `::1`, so anything looking for the editor at `127.0.0.1` — a test harness, a health check, a script — times out against a server that is demonstrably running and printing its URL. **Fix/policy:** set `server.host` to `127.0.0.1` explicitly, the same literal the sidecar binds to. Spell it as an address, never as `localhost`, anywhere the two halves have to find each other. _[earned 2026-08-11, Vite 8.2.1]_

### UG4: Panels have close buttons and the kernel has nowhere to reopen them from

Dockview tabs render a close affordance by default, and the shell has no panel menu and no layout persistence, so a closed panel is gone until the page is reloaded. Reloading restores the default layout, which makes this shallow rather than dangerous — but it is a real dead end for a human who does not know that. **Fix/policy:** it is fixed by the feature that adds a panel menu plus saved layouts, not by hiding the button; hiding a control while its keyboard and context-menu paths still work is worse than leaving it visible. Until then, say so in the hand-off. _[earned 2026-08-11]_

## Contracts

Contracts are referenced as file paths, never paraphrased as prose. Read the file; don't trust a summary of it.

- `kernel-2d/editor/shell/panels.tsx` — every panel the editor has and the layout it opens in. Adding a panel happens here and nowhere else (U1).
- `kernel-2d/editor/shell/App.tsx` — the shell: status strip above, docking layout below.
- `kernel-2d/editor/shell/useSidecarStatus.ts` — how the editor learns which project it is connected to, and what it does when the answer stops coming (U3).
- `kernel-2d/editor/shell/StatusStrip.tsx` — the connection line, including the `data-testid` hooks the browser suite reads.
- `kernel-2d/editor/shell/shell.css` — the frame around the docking layout. Dockview's own chrome comes from its theme, not from here.
- `kernel-2d/vite.config.ts` — the editor's root folder, the `/api` proxy to the sidecar (U2), and the loopback binding (UG3).
- `kernel-2d/scripts/editor-server.ts` — the editor window's host, port, and open-a-browser knobs, with their environment variable names.
- `kernel-2d/tsconfig.editor.json` and `kernel-2d/tsconfig.base.json` — the browser half of U4.

**Not yet written** — until a path appears here, the contract does not exist and must not be assumed:

- The Zustand/immer store and the transaction API it fronts. No document state exists yet, so neither dependency has been added.
- Inspector auto-generation from Zod schemas.
- The panel menu and saved layouts (UG4).
- Any keyboard shortcut registry.
