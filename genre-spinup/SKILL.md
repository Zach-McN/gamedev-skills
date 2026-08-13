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

### S2: Knowledge true of one game lives in that game's `.claude/skills/`, not in this library

A game will accumulate conventions with real authority over sessions — how its bosses are phased, what its encounter tables mean, which invariants its world structure holds — that would be *wrong* in any other game. Putting them here would pollute the universal library; leaving them unwritten repeats the pre-skill failure this whole system exists to prevent.

So: `MyGame/.claude/skills/<domain>/SKILL.md`, same three registers, same dual-write obligation, same "earned, never invented" standard. Claude Code loads project skills by their `description`, so a well-described local skill self-selects when its subsystem is touched. _[settled 2026-08-12, unexercised until the first game — expect the first spin-up to amend this]_

## Gotchas

_None recorded yet. Filled via dual-write as genres are spun up._

## Contracts

_None recorded yet. Reference kernel-repo and genre-repo file paths here; never paraphrase them as prose._

**Deliberately unbuilt** — each has a named trigger, and building it before the trigger is the seeded-content failure the first parity drill measured (a rule written ahead of its referent goes stale invisibly):

- A boundary test asserting the kernel never imports from `games/`. One line in `tests/architecture/boundaries.test.ts` **the day a `games/` folder exists** — written earlier it scans a missing folder and passes vacuously (`editor-verification` W9).
- A `PreToolUse` hook refusing edits to game-content files that lack a `generatedBy` marker. Built **when the first game has human-authored content to protect** — an unexercisable hook can only misfire, and a hook that misfires trains its owner to override it.
- A workspace-root `CLAUDE.md` above kernel, games and skills. Written **when a `games/` folder makes a session's working set span repos** — its content is whatever the first spin-up session actually needed told at the top, not what one could guess today.
