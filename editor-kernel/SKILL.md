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

### D3: All authored state is human-readable text in the project folder

No binary project database, no hidden index of record. **Reason:** three compounding payoffs — git-diffable history, and the ability for a session to inspect live game state with `grep` instead of loading it through the editor. The third is context economics: a well-factored text project lets a session read the schema rather than the codebase, which is what keeps sessions cheap as the kernel grows. _[seeded 2026-08-11, report §3/§4]_

### D4: Sidecar `.meta` files; the asset browser mirrors the folder 1:1

The binary lands wherever the human put it from Photoshop or Blender. A JSON sidecar beside it (`knight.png` + `knight.png.meta`) holds import settings: slicing, pivot, filtering, collider generation. No import step, no copy, no rename. **Reason:** the folder *is* the database. The editor's only privilege over the human's files is annotation — the moment it starts moving or rewriting them, the 1:1 guarantee dies and the human loses the ability to work in their own tools without the editor's permission. _[seeded 2026-08-11, report §4/§8]_

### D5: References carry both a stable ID and a human-readable path

`"sprite": {"id": "a3f9", "path": "assets/textures/knight.png"}`. IDs are generated once and stored; a fixup tool reconciles the pair when files move. **Reason:** the two properties are irreconcilable in a single field. IDs survive renames but make files unreadable; paths stay greppable and let a session understand a scene by reading it, but break on every move. Carrying both costs a few bytes and removes the tradeoff. _[seeded 2026-08-11, report §4]_

### D6: Every format is a Zod schema, and the schema file is the single source of truth

The runtime validates on load, the editor validates on save, and both read the same definition. Adding a component type changes the schema in exactly one place; `tsc` and the validators then reveal every tool that needs updating. **Reason:** this is the primary defense against serialization drift — the save format and load format quietly disagreeing — which is the most common failure mode of AI-built editors. See G1. _[seeded 2026-08-11, report §4]_

### D7: Undo is document-level, via immer patches through a transaction API — never per-tool

All mutations go through the kernel's transaction API. Undo is implemented once as patches over the in-memory document, not as per-tool inverse commands. **Reason:** per-tool undo is the #1 source of editor jank, because every new tool is a fresh chance to write an inverse operation that is subtly wrong, and the bug surfaces three actions later where nobody is looking. Document-level undo means every future genre tool inherits correct undo for free, having written no undo code at all. This decision is load-bearing enough that a session bypassing the transaction API is a defect regardless of whether its feature works. _[seeded 2026-08-11, report §5/§13]_

### D8: A Node sidecar owns the filesystem; the editor is a local web app

Chokidar watcher + REST for JSON read/write + WebSocket for change events + static asset serving. **Reason:** browsers cannot watch folders. The File System Access API is Chromium-only, permission-prompty, and cannot push change events — so the "save a PNG and watch it appear" workflow is impossible in-browser and trivial with a small Node process. _[seeded 2026-08-11, report §8]_

### D9: One command starts everything

`npm run editor` starts Vite and the sidecar concurrently. **Reason:** the editor must be ordinary software that runs without AI from day one. If starting it requires a session, the human cannot author content on their own schedule, and the whole division of labor stops working. _[seeded 2026-08-11, report §8]_

### D10: Sidecar first, Tauri later

Ship the web-app-plus-sidecar shape; wrap in Tauri only when a genre editor matures and wants to be double-clickable, swapping the sidecar's HTTP calls for Tauri's fs/watch APIs behind the same interface. **Reason:** the sidecar keeps the dev loop inspectable — Playwright drives it, `curl` pokes it, every boundary is observable. Deferring the wrapper costs nothing because the interface is the same; adopting it early costs verification. _[seeded 2026-08-11, report §8]_

### D11: The kernel persists; the genre layer regenerates

The kernel is the genre-agnostic ~60%: scaffold, sidecar, asset browser, meta system, serialization core, undo/redo, inspector framework, play-in-editor harness, test setup. Genre tooling is built fresh per game. **Reason:** undo and serialization are precisely the systems that must never be rewritten ad hoc, because that is where subtle bugs live. Genre tools are the opposite — a bespoke wave designer beats a generic timeline, and each is small enough to produce quickly against established kernel patterns. _[seeded 2026-08-11, report §5]_

### D12: The skills are the engine; the kernel repo is their cached output; the tests are the contract

Dual-write is definition-of-done, and the parity drill (regenerate from skills alone, run the real suite against it) is the mechanical proof the skills have not silently fallen behind. **Reason:** the two pure strategies both fail — maintaining the kernel forever ossifies it, and regenerating from nothing re-pays the hard-systems tax every game. Holding both requires that the knowledge live somewhere the code cannot drift away from unnoticed. Costs ~10–15% overhead per session. _[seeded 2026-08-11, report §5]_

### D13: Export targets are kernel-level, one command each

`npm run export:web` (Vite static build) and `npm run export:desktop` (native shell). **Reason:** the runtime is web-native, so both targets are a packaging step over identical game code — which makes the desktop-shell choice deferrable and reversible. Building both into the kernel means every genre editor inherits them instead of re-deciding. _[seeded 2026-08-11, report §4]_

### D14: AI writes code that writes files; only humans author data through the tools

The narrow exception is migration scripts — a one-off format converter is tooling, not authorship. **Reason:** the moment level data is authored by prompt rather than by tool, the data stops reflecting human intent and the anti-slop guarantee is gone. This is the thesis of the whole methodology, not a preference. _[seeded 2026-08-11, report §3/§9]_

### D15: One editor UI stack across both dimensions

React + docking layout + Zustand/immer + Zod, with only the viewport differing between 2D and 3D kernels. **Reason:** the panels, trees, and inspectors are identical work in both; sharing the stack means the 3D kernel inherits a proven shell and the hard 3D-specific problems (camera, raycast selection, gizmos) get the full budget. Detailed idioms belong to `editor-ui`; the constitution only fixes the stack. _[seeded 2026-08-11, report §6/§7]_

---

## Gotchas

### G1: Serialization drift is the top structural risk, and it must be fenced before the second tool exists

Save and load quietly disagree; nothing errors; corruption surfaces later as lost work. **Fix/policy:** the Zod-single-source-of-truth rule (D6) plus a standing `load(save(x)) === x` round-trip test for every schema, running on every change. The ordering matters — the tripwire must exist before the second tool is written, because the drift becomes expensive exactly when there are multiple writers of the same format. _[seeded 2026-08-11, report §4/§13]_

### G2: A tool that mutates the document outside the transaction API appears to work

Nothing fails at write time. The damage shows up as undo skipping a step or restoring stale state, far from the tool that caused it. **Fix/policy:** treat "does this mutation go through the transaction API?" as a review question for every session that touches the document, and prefer making the direct-mutation path structurally unavailable over documenting that it shouldn't be used. _[seeded 2026-08-11, report §5/§13]_

### G3: Quality erodes invisibly when nobody reads the code

Duplication, dead code, and creeping complexity are not caught by behavior tests — the editor keeps working while future sessions get slower. **Fix/policy:** a scheduled gardening session every few features (audit the codebase against the skills' conventions, second-model review, refactors behind the same green gate). The smoke alarm is quantitative: if session cost or time for similar-sized features trends upward, the garden needs tending. _[seeded 2026-08-11, report §13]_

### G4: "Just generate this one level" breaks the guarantee retroactively

The pressure appears when a tool doesn't exist yet and the content is needed now. **Fix/policy:** the rule is absolute (D14) — the answer to missing tooling is the tool, not the content. Note the failure is retroactive: once prompt-authored data is in the project, no later tool work restores the provenance of what shipped. _[seeded 2026-08-11, report §3]_

### G5: Scope creep toward "building an engine" has no natural stopping point

**Fix/policy:** the genre spec is the fence — if a tool isn't justified by a noun in the spec, it isn't built this cycle. The kernel will become an engine over time; the requirement is that it happen by accretion from shipped needs rather than by anticipation. _[seeded 2026-08-11, report §13]_

---

## Contracts

Contracts are referenced as file paths, never paraphrased as prose. Read the file; don't trust a summary of it.

**Exists now:**

- `kernel-2d/CLAUDE.md` — project law: division of labor, AI-generated-content marking rules, folder map, definition of done, session conduct. Outranks this skill on process; this skill outranks it on architecture rationale.
- `kernel-2d/docs/ai-game-tooling-report.md` — the source these decisions were seeded from. §4 architecture, §5 kernel/genre split and Option C, §8 sidecar shape, §9 the vibe-coding contract.
- `gamedev-skills/vendor/phaser4/` — vendored Phaser 4 ground truth. Owned by `phaser4-runtime`; referenced here only so the kernel's runtime layer knows where authority lives.

**Not yet written** — these are the kernel's core contracts and land as the corresponding sessions build them. Until a path appears here, the contract does not exist and must not be assumed:

- `SceneSchema` — entity list, component maps, transforms, asset refs.
- `PrefabSchema` — reusable entity templates.
- `MetaSchema` — the `.meta` sidecar format.
- The transaction API surface — the single entry point for all document mutation (D7).
- The sidecar REST/WebSocket API — read/write and change-event shapes (D8).
