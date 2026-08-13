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

So: `MyGame/.claude/skills/<domain>/SKILL.md`, same three registers, same dual-write obligation, same "earned, never invented" standard. Claude Code loads project skills by their `description`, so a well-described local skill self-selects when its subsystem is touched. _[settled 2026-08-12]_

**Amended by the first spin-up, and the amendment is about what makes the empty folder work rather than about the rule.** `games/tower-defense/.claude/skills/` was created empty, and an empty folder is the correct starting state — but it only functions as S2 intends because the *shared* library reaches a session by a different route (S3). Had the game folder's own `skills/` been the only skills directory in its tree, a session working in the game would have had the game's local knowledge and none of the library that built the kernel underneath it. The two are complements, and the empty folder is only safe once the root junction exists. _[amended 2026-08-13, first spin-up]_

### S3: The workspace root gets a map and a second skills junction — the map says which `CLAUDE.md` wins, not what the rules are

Fired 2026-08-13 by the first spin-up, which is the trigger this was parked on: a `games/` folder makes a session's working set span repos.

**What it actually had to say**, as against what one might have guessed. The guess would have been a rulebook. What the session needed was a *map* and one disambiguation:

- The three folders are three different kinds of thing — `kernel-2d` is the application, `games/*` are documents it opens, `gamedev-skills` is the library that regenerates both. Naming that is most of the value; the commonest mistake available here is editing the right content in the wrong repo.
- **Which `CLAUDE.md` governs.** Three of them are now in scope at once, and without a stated order the root file reads as the senior one. It is not: `kernel-2d/CLAUDE.md` is the working constitution and the nearest file wins. Say so at the root, or the root file quietly acquires authority it was never meant to have.
- That every touched repo must end clean and committed — because the definition of done lives in the kernel's file, and a session that never opens the kernel would otherwise not meet it.

**The root is not a git repo, so the file is version-controlled by nothing.** Recorded rather than fixed: making the root a repo would nest three repos inside a fourth, and the alternative — parking it in `gamedev-skills` behind a junction — buys history for the one file least likely to change. Named in the file itself so nobody mistakes it for a durable record.

**The skills junction goes at the root *and* stays in the kernel.** `gamedev-skills` is junctioned at `gamedev/.claude/skills` and at `kernel-2d/.claude/skills`. The cost is real and was accepted deliberately: a session opened at the root discovers all seven skills twice, once workspace-wide and once scoped to `kernel-2d/`. The alternative — one junction at the root — means `kernel-2d` opened directly, which is how most kernel sessions start, loads no skills at all. Duplicate discovery is noise; no discovery is the paradigm not working. _[earned 2026-08-13, first spin-up]_

## Gotchas

### SG1: A game folder is a git repo, so the Assets panel shows the human tooling files

The first game folder opened in the editor listed `.claude/`, `.claude/skills/`, `.gitattributes` and `.gitignore` in the Assets panel alongside `assets/` and `scenes/`, and offered `.claude/skills` as a destination in the move-folder picker. Nothing is broken — the sidecar's ignore list (`kernel-2d/sidecar/ignore.ts`) is `node_modules`, `.git`, `dist` and OS junk, and deliberately excludes "anything the human might have authored". Dotfiles were simply never in a project folder before, because every project until now was a scratch folder made by `sample-project`.

It also affects the empty-folder problem: `scenes/`, `prefabs/`, `data/` and the rest only survive a clone if they hold a `.gitkeep`, and each of those shows in the panel too.

**Fix/policy:** leave it until the human says it is noise — the ignore list's stated principle is to hide tooling and never content, and a dotfile rule is a one-line change to `IGNORED_NAMES` when it is wanted. What must *not* happen is hiding them by putting the game's tooling somewhere outside the project folder; the folder the editor watches and the folder git tracks are the same folder on purpose. _[earned 2026-08-13, first spin-up]_

### SG2: This skill's own estimate of the boundary test was wrong, and the way it was wrong is the lesson

It was parked below as "one line in `tests/architecture/boundaries.test.ts`" — meaning one more entry in that file's `FORBIDDEN` list, named `games`. **That line could never have fired.** The existing check resolves an import and takes the first path segment *relative to the kernel repo root*; for anything outside the repo that segment is `..`, never a folder name. `games/` is a sibling of the kernel repo, not a folder inside it, and was invisible to a test written that way. The mechanism and the general form of the fix are `editor-verification` W20.

Two things worth carrying past this one instance:

1. **The estimate was made by pattern-matching an existing test rather than by reading it.** "One more line in the list" was true of the *shape* of that file and false about what the list could see. An unbuilt item's cost estimate is a guess about code nobody re-read.
2. **The requirement was also wider than the estimate assumed.** "The kernel never imports from `games/`" is about the whole kernel; the existing block scans only `runtime/`, because it is about what *ships*. An editor panel reaching into a game folder is the same mistake and would not have shown up. Two boundaries, two blocks, and the repo one scans all four layers.

_[earned 2026-08-13, first spin-up]_

## Contracts

- `games/tower-defense/docs/GENRE-SPEC.md` — the first genre spec, and the shape the rest should follow: what the game is, what the player does, the nouns, and an explicit **Not in this game** list. The last section is what makes it a fence rather than a wish list — G5 can only refuse something if the spec is willing to say no.
- `games/tower-defense/CLAUDE.md` — the game-folder template: the ownership split, the fence, the local-skills rule, and a named statement of what does not run yet.
- `gamedev/CLAUDE.md` — the workspace map (S3).
- `kernel-2d/tests/architecture/boundaries.test.ts` — both boundaries as tests: the runtime/editor one (`editor-kernel` D20), and the repo one that keeps the kernel out of `games/` (SG2).

**Deliberately unbuilt** — each has a named trigger, and building it before the trigger is the seeded-content failure the first parity drill measured (a rule written ahead of its referent goes stale invisibly):

- **Loading a game folder's `src/` so it can supply systems.** Parked deliberately on 2026-08-13, when the update loop landed, and it is the **first thing the first game session should take up** — everything else on this list waits for a genre spec; this waits for nothing but a decision. What exists already: `runLevel` takes its systems as an argument, so the seam is shaped, and `BUILT_IN_SYSTEMS` is the list the editor and an export both start from. What does not exist, and what has to be decided rather than guessed:
  1. **How a game folder's TypeScript is compiled into both surfaces.** The editor is a Vite dev server rooted in `kernel-2d`; an export is a bundle produced by the export command. A game's `src/` sits in a folder neither of them currently reaches, and the answer has to be one answer — two build paths is the shipped game and the editor's Play running different code, which is D2's failure with a new fuse.
  2. **How a level says which systems it wants** — or whether it does not, and a project runs all of its own. The second is smaller and is probably right first; it is a decision because per-level system lists are the sort of thing that is very hard to take back out.
  3. **What happens when that code changes while the editor is open.** Everything else in this kernel reloads within the second (D18/D19). Code has no obvious equivalent, and "press Play again" may well be the honest answer — but it should be an answer somebody chose.

  The reason it was parked rather than half-built: it is a build-system question wearing a runtime question's clothes, and the shape of the answer depends on what the first game's folder actually looks like. Building it against an imagined game folder is precisely S1.

- A `PreToolUse` hook refusing edits to game-content files that lack a `generatedBy` marker. Built **when the first game has human-authored content to protect** — an unexercisable hook can only misfire, and a hook that misfires trains its owner to override it. Still unbuilt after the first spin-up, and the trigger is worth restating now that it is close: `games/tower-defense/` exists but holds no human-authored content, so the hook would still have nothing to guard. It fires the day Zach puts a texture or a level in that folder.

**Two of these fired on 2026-08-13** and have moved into the registers above rather than being deleted, because what they turned out to cost is the part worth keeping:

- The boundary test → Contracts, and SG2 for how the estimate was wrong.
- The workspace-root `CLAUDE.md` → S3.
