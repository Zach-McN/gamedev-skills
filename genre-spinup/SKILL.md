---
name: genre-spinup
description: The genre spin-up playbook — the repeatable process for standing up a new game genre on top of the editor kernel and runtime. Consult this skill whenever starting a new genre, deciding what a genre editor needs beyond the kernel, or capturing what was learned spinning one up so the next genre goes faster. Feeds the per-genre `genre-*` skills.
---

# genre-spinup

The playbook for spinning up a new game genre on top of the kernel and runtime. This skill captures the repeatable process — the steps, decisions, and pitfalls of taking the shared editor foundation and specializing it for a specific genre — so that each new genre is faster and more predictable than the last. What is learned spinning up a genre also seeds that genre's own `genre-*` skill. Content is earned from real spin-up sessions, never invented.

## Decisions

### S1: A genre tool lives in the game folder, and the kernel grows by extraction from it — never by anticipation of it

Settled 2026-08-12, before the first spin-up, by judging an external proposal against the library's earned entries rather than by experience — recorded now because the rejected alternative is the *obvious* design and will be proposed again.

**The rule.** A bespoke tool (a wave designer, a boss-phase editor, a dungeon-connectivity view) is built in the game's own folder, speaking the game's own vocabulary. When — and only when — a *second* game wants the same underlying primitive, the smallest reusable piece is extracted into the kernel and both games import it. That is D20's move (`editor-kernel`), which has fired three times and is mechanical each time.

**The rejected alternative, named so it stays rejected:** pre-building a generic layer — a kernel `Timeline<T>`, an `editor-sdk/` with `registerPanel`-style extension points — for game tools to specialize. Three earned entries already say why:

- `editor-kernel` D11: *"a bespoke wave designer beats a generic timeline"* — the constitution's own words.
- `editor-ui` U28: a component shared before a second owner exists *"is a guess about what two would have in common."*
- `editor-kernel` D15's confirmation: machinery built before its first consumer *"fixes its shape against an imagined document"* — the reason zustand was not installed with the shell.

The test for kernel admission stays G5's, which is checkable: **is the tool justified by a noun in the genre spec?** Not the hypothetical-second-game test, which can be argued either way forever.

**The same promotion rule covers knowledge.** Knowledge true of one game only goes in that game's own skills (see S2); it is promoted into `gamedev-skills` when a second game proves the general part general — the code rule and the knowledge rule are one rule.

**Amended by the first thing that had to break it, and the exception is narrower than it looks: a *seam* may be built ahead of its consumer, but it has to be built against a real one.**

The kernel gained an update loop before any game existed (`editor-kernel` D27). A runner with no systems is untestable in the way that matters — nothing proves the interface can carry a real behaviour, that a level's data reaches it, that Stop puts everything back, or that an export runs the same code. So the kernel ships **one** system, `spin`, with a component in the scene registry to drive it, placed on the sample project's health icon.

Four conditions, and they are what keep this from being a licence:

1. **One, not a family.** Not spin *and* drift *and* an oscillator, because the second one is no longer serving the seam — it is a genre feature with a seam as its excuse. There is a test asserting the built-in list has exactly one entry, so growing it is a decision somebody makes on purpose.
2. **Chosen to be unmistakably scaffolding.** A turn rate is not a mechanic anybody would design a game around. Movement, health or damage would each have been read as the kernel taking a position on what games are.
3. **The consumer is real, not a fixture.** The sample project is what the browser suite runs and what the export command exports, so the seam is exercised end to end by the same content a human opens. A test-only entity would have proved the interface compiles and nothing else.
4. **Its removal is named and cheap.** `spin` and `runtime/game/systems/spin.ts` leave together the day a game folder's own code can supply systems, and nothing else in the kernel changes. That is the check that the seam was drawn in the right place — if taking the demonstration out would break the machinery, the machinery was shaped around the demonstration.

The rule to carry forward: **S1 forbids anticipating a genre's *tools*; it does not forbid a seam having exactly one honest consumer.** The tell for the difference is condition 4 — if you cannot say what removing it would cost, you have built the thing S1 is about. _[earned 2026-08-13, first exception]_

### S2: Knowledge true of one game lives in that game's `.claude/skills/`, not in this library

A game will accumulate conventions with real authority over sessions — how its bosses are phased, what its encounter tables mean, which invariants its world structure holds — that would be *wrong* in any other game. Putting them here would pollute the universal library; leaving them unwritten repeats the pre-skill failure this whole system exists to prevent.

So: `MyGame/.claude/skills/<domain>/SKILL.md`, same three registers, same dual-write obligation, same "earned, never invented" standard. Claude Code loads project skills by their `description`, so a well-described local skill self-selects when its subsystem is touched. _[settled 2026-08-12, unexercised until the first game — expect the first spin-up to amend this]_

## Gotchas

_None recorded yet. Filled via dual-write as genres are spun up._

## Contracts

_None recorded yet. Reference kernel-repo and genre-repo file paths here; never paraphrase them as prose._

**Deliberately unbuilt** — each has a named trigger, and building it before the trigger is the seeded-content failure the first parity drill measured (a rule written ahead of its referent goes stale invisibly):

- **Loading a game folder's `src/` so it can supply systems.** Parked deliberately on 2026-08-13, when the update loop landed, and it is the **first thing the first game session should take up** — everything else on this list waits for a genre spec; this waits for nothing but a decision. What exists already: `runLevel` takes its systems as an argument, so the seam is shaped, and `BUILT_IN_SYSTEMS` is the list the editor and an export both start from. What does not exist, and what has to be decided rather than guessed:
  1. **How a game folder's TypeScript is compiled into both surfaces.** The editor is a Vite dev server rooted in `kernel-2d`; an export is a bundle produced by the export command. A game's `src/` sits in a folder neither of them currently reaches, and the answer has to be one answer — two build paths is the shipped game and the editor's Play running different code, which is D2's failure with a new fuse.
  2. **How a level says which systems it wants** — or whether it does not, and a project runs all of its own. The second is smaller and is probably right first; it is a decision because per-level system lists are the sort of thing that is very hard to take back out.
  3. **What happens when that code changes while the editor is open.** Everything else in this kernel reloads within the second (D18/D19). Code has no obvious equivalent, and "press Play again" may well be the honest answer — but it should be an answer somebody chose.

  The reason it was parked rather than half-built: it is a build-system question wearing a runtime question's clothes, and the shape of the answer depends on what the first game's folder actually looks like. Building it against an imagined game folder is precisely S1.

- A boundary test asserting the kernel never imports from `games/`. One line in `tests/architecture/boundaries.test.ts` **the day a `games/` folder exists** — written earlier it scans a missing folder and passes vacuously (`editor-verification` W9).
- A `PreToolUse` hook refusing edits to game-content files that lack a `generatedBy` marker. Built **when the first game has human-authored content to protect** — an unexercisable hook can only misfire, and a hook that misfires trains its owner to override it.
- A workspace-root `CLAUDE.md` above kernel, games and skills. Written **when a `games/` folder makes a session's working set span repos** — its content is whatever the first spin-up session actually needed told at the top, not what one could guess today.
