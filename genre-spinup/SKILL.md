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

### S4: The first game's own code is what finally lets the kernel's scaffolding leave, and the swap is a demotion rather than a deletion

S1's exception let the kernel ship one system, `spin`, on four conditions, the fourth being that its removal be named and cheap. The day arrived on 2026-08-13, and the move that made it cheap is worth recording because it is not the obvious one.

**The obvious move is to delete the demo. The right move was to demote it.** `spin` stayed exactly where it was; what changed is that the kernel stopped *running* it. `BUILT_IN_SYSTEMS` went from being what every level ran to being a menu nothing reads, and the sample project's own `src/systems/index.ts` imports `spinSystem` by name and lists it alongside a system of its own. The health icon still turns — because content asks for it, not because the engine imposes it.

Three things fall out, and the third is the one to carry forward:

1. **Removability became checkable rather than promised.** Emptying `BUILT_IN_SYSTEMS` breaks exactly one test — the one asserting it contains `spin` — and every browser test still passes, health icon included. That is condition 4 satisfied by running it rather than by arguing it.
2. **The sample project became the seam's real consumer**, which is what S1's condition 3 wanted all along. The browser suite points at the sample, so the mechanism is exercised end to end by the same content a human opens, through the real generator rather than a test-only file.
3. **A demonstration leaves in two steps, not one.** First it stops being load-bearing; then it is deleted. Doing both at once means the deletion and the rewiring fail together and neither is diagnosable. The actual deletion — the system, its component in the scene format, its Inspector field, the sample's use of it — is a separate session with nothing left depending on it.

**What it cost against the estimate.** The parked item named three sub-decisions and they were the right three; none turned out to be a trap. The unbudgeted work was elsewhere and in two places: the generator needed a third marking mode, because a `.ts` file can carry `generatedBy` neither as JSON nor in a `.meta` beside it without the asset pipeline annotating something that is not an asset; and the dev server does not watch a folder outside its root, which fails as *silence* (`editor-kernel` G14). Both are the same shape — **the mechanism was fine and the things around it had assumed every file was content.** _[earned 2026-08-13, first game code]_

### S5: A game folder gets its own test runner, the first time its own code has logic in it

Fired 2026-08-13 by the first real feature in the first game — monsters walking a
drawn road. Until then a game's code was two lines that could be read and believed;
the moment it had to work out an *order* from geometry, believing it was no longer
available.

**The kernel's suite cannot be the answer, by construction.** `boundaries.test.ts`
asserts that no relative import escapes the kernel repo (SG2), so a kernel test
cannot reach into a game folder — and pointing the kernel's Vitest include at a
sibling would weld `npm test` in the application to the presence of a document. Both
directions are the same mistake in different clothes.

So the game folder gets a `package.json` and its own Vitest, pinned to the same
version the kernel uses so both are tested by one runner rather than two. Three
things fell out that a next genre should expect:

1. **`node_modules` in a content folder is already handled** — the sidecar's ignore
   list has it, and the game's `.gitignore` had it before there was anything to
   ignore. Nothing had to change.
2. **`package.json` shows in the Assets panel** and the Inspector calls it a document
   it cannot read. That is SG1's noise with one more file in it, not a new problem:
   `tsconfig.json` was already sitting there being exactly as unreadable.
3. **The tests take entity lists, not files.** A system is handed entities and a step
   size and nothing else, so a fixture that went through JSON would be testing the
   kernel's loader on the way past — which the kernel already tests, and which turns
   one failure into two possible faults.

**The general rule:** the application is tested where it lives and so is the document;
a test that has to span both is a sign the seam between them is in the wrong place.
_[earned 2026-08-13, first game feature]_

### S6: Content a human will replace comes from a throwaway generator; content that must be re-runnable gets a committed one

The kernel's `sample-project` generator is committed and produces identical bytes
every run, because a scratch project is made over and over by whoever is trying the
editor (`editor-kernel` G4). The first game's art, levels and prefabs are the
opposite kind of thing: **scaffolding authored once, for the human to replace piece
by piece.** A committed generator for those would be a maintenance burden attached to
files whose whole purpose is to stop existing — and, worse, one living in the kernel
that knew what a road tile was, which is S1's failure arriving through the back door.

So it was a script in a scratch folder, run once, kept nowhere. What makes that safe
rather than sloppy is the two conditions it met:

- **It went through the kernel's own schemas and serializers** rather than
  hand-writing JSON, so what landed on disk was valid by construction. This is the
  marking rules' "prefer the tool path" spent where it actually pays.
- **It validated its own output before writing**, and the level was then loaded back
  through the real `loadScene` and stepped by the real system. A generator that
  writes a file the editor cannot open looks exactly like one that worked.

The record of what produced the content is the `generatedBy` marker on every file,
which is the thing that has to survive — not the script. _[earned 2026-08-13, first
game content]_

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
- `games/tower-defense/package.json` and `games/tower-defense/tests/levels.ts` — the shape of a game folder's own verification (S5): its own runner, pinned to the kernel's version, and fixtures built as entity lists rather than as files.
- `games/tower-defense/docs/authoring.md` — the game-folder counterpart to `kernel-2d/docs/using-the-editor.md`. The kernel's page says what the *editor* can do; this one says how to author *this game* with it, and a genre needs one as soon as its levels have vocabulary of their own that the Inspector cannot show.

**Deliberately unbuilt** — each has a named trigger, and building it before the trigger is the seeded-content failure the first parity drill measured (a rule written ahead of its referent goes stale invisibly).

**An entry here carries a trigger and an intent, and deliberately not an implementation.** That is a correction, applied 2026-08-13 after both entries that fired came with a sketch of the work and both sketches were wrong (SG2). The sketches were not merely inaccurate, they were *misleading in a specific direction*: each was made by pattern-matching the shape of code nobody re-read, so each read as a cost estimate while actually being a guess, and "one line in `boundaries.test.ts`" would have been built as one line that could never have fired. A parked item's job is to make sure the thing is not forgotten and not built early. Sizing it is the job of the session that has the file open.

- A `PreToolUse` hook refusing edits to game-content files that lack a `generatedBy` marker.
  - **Trigger:** the first time a game folder holds human-authored content — the day Zach puts a texture or a level into `games/tower-defense/`. It has not fired: the folder exists and everything in it is generated.
  - **Intent:** make "never overwrite human work" structural rather than a rule sessions are trusted to remember. An unexercisable hook can only misfire, and a hook that misfires trains its owner to override it.
  - **Known constraint, because it is the kind that decides the design:** the marker now lives in three places depending on what the file can hold — inside a JSON document, in the `.meta` beside a binary, in a comment in a source file. Anything checking for it must know all three.

- A way for a game folder to put an authoring surface of its own in front of the human.
  - **Trigger:** the first genre tool that cannot be done with the kernel's existing gestures. It has *not* fired yet, and the near miss is the useful part of this entry: drawing a road was expected to need a tile painter and turned out to be reachable by placing a prefab twenty-six times, because S1's rule sent the work at the game's own vocabulary rather than at a panel. The trigger is a tool that is genuinely impossible this way, not one that is merely tedious — tedium is answered by the kernel's own placement gestures getting better, which is genre-neutral and needs none of this.

    **The second half of that sentence was paid out on 2026-08-13 and it held.** The tedium — twenty-six careful drags — was answered in the kernel by a settable snap and a mode that places on every click (`editor-ui` U31), about two hundred lines, no panel, and nothing in either of them that knows what a road is. A road is now clicked out in twenty-six presses that cannot land wrong. This entry is still unfired, and it is now *harder* to fire, which is the outcome the rule was aiming at: every gesture the kernel gains is a genre tool that turns out not to be needed. Worth knowing before estimating the next one — the first instinct ("this needs a painter") was wrong twice about the same feature.
  - **Intent:** answer the question S1 has been deferring — a game folder can supply *systems* to the runtime through one compiled seam, and can supply *nothing* to the editor. Whether that asymmetry is correct is undecided, and S1's rejection of a speculative `registerPanel` extension API is not the same as an answer.
  - **Known constraint, because it is what makes this hard rather than long:** the kernel imports nothing from a game folder and that is a test, not a convention (SG2). Whatever this turns out to be, it cannot be the editor reaching sideways into `games/`.

**Three of these fired on 2026-08-13** and have moved into the registers above rather than being deleted, because what they turned out to cost is the part worth keeping:

- The boundary test → Contracts, and SG2 for how the estimate was wrong.
- The workspace-root `CLAUDE.md` → S3.
- **Loading a game folder's `src/` → `editor-kernel` D28**, which is where it belongs: it turned out to be an architecture decision about the kernel rather than a fact about spinning genres up. All three of its named sub-decisions were answered rather than guessed — one plugin for both surfaces, a project running all of its own systems with no per-level list, and Play re-reading the code with no page reload. S4 records what the *shape* of the answer cost, which is this skill's business.
