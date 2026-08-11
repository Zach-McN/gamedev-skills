CLAUDE.md — gamedev-skills
This repo is a skill library for AI-built game editors. It is the permanent asset of the whole methodology: the kernel repos (`kernel-2d`, `kernel-3d`) and game projects are regenerable outputs; this library is the engine that regenerates them. Treat every edit here with the same rigor as production code. Full methodology: see `docs/ai-game-tooling-report.md` if present, or ask.
What lives here
One folder per skill, each with a `SKILL.md`:

* `editor-kernel` — the architecture constitution
* `text-formats` — the JSON/Zod format playbook
* `editor-ui` — React/docking/inspector idioms
* `editor-verification` — testing recipes and invariants
* `phaser4-runtime` — Phaser 4 specifics + v3-contamination warnings
* `threejs-runtime` — Three.js/WebGPU (stub until the 3D kernel begins)
* `genre-spinup` — the genre spin-up playbook
* `genre-*` — one per shipped genre, added over time

Plus `vendor/` — ground-truth library docs (Phaser 4 .d.ts, changelogs, migration notes). Vendored docs are read-only reference; never edit them, only replace them wholesale when versions change.
Rules for editing skills

1. No invented content. Skills contain only knowledge earned from real sessions — real decisions, real gotchas, real contracts. Never pad a skill with plausible generic advice. A thin, true skill beats a thick, speculative one. If asked to scaffold or expand a skill, write structure, not filler.
2. Three registers, kept distinct in every skill:
   * Decisions — what was chosen and why (e.g. "document-level undo via immer patches, because per-tool undo is the #1 source of editor jank")
   * Gotchas — accumulated traps and their fixes
   * Contracts — schemas and API shapes, referenced as files (paths into the kernel repo or vendored docs), never paraphrased as prose
3. Skills must regenerate, not describe. The test of a skill is: could a fresh session rebuild the corresponding code from this file plus the test suite? Write for that reader. Prose alone regenerates vibes; decisions + contracts + tests regenerate the kernel.
4. Version awareness. Skills reference pinned library versions (e.g. "targets Phaser 4.1.x"). When a skill's guidance is version-dependent, say so explicitly.
5. The human never reads code, but does read skills. Write SKILL.md files in clear language a designer can skim. Code snippets are allowed where they are the contract itself.

Session conduct

* Git: commit before and after every session.
* Most edits to this repo arrive as dual-writes from kernel/game sessions (via the symlink). Sessions opened directly here are for: scaffolding, reorganizing, vendoring docs, parity-drill harvesting, and release prep.
* During parity-drill harvesting: every drill failure maps to either a missing piece of skill knowledge (fix the skill) or a test asserting an accident (flag it — the test fix happens in the kernel repo, not here).
* Never delete a gotcha because it seems obsolete; mark it superseded with the version/date instead. Gotchas are the repo's most expensive content.
