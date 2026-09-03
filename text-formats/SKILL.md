---
name: text-formats
description: The JSON/Zod format playbook for game-editor documents — how on-disk and in-memory data formats are defined, validated, versioned, and migrated. Consult this skill whenever designing or changing a document format, writing a Zod schema for editor data, handling format versioning or migration, or deciding how the editor serializes and parses its files.
---

# text-formats

The playbook for the editor's text data formats — the JSON shapes that editor documents take on disk and the Zod schemas that validate them. This skill governs how formats are structured, validated, versioned, and migrated over time so that documents written by one version of the editor stay readable by the next. Content is earned from real format-design sessions, never invented.

Targets Zod 4.x.

## Decisions

### T1: Every document carries a `format` string and a `version` number, both as literals in the schema

An unknown version fails to parse loudly rather than being read on a hope. **Reason:** the alternative is a document from a future editor being partially understood by an older one, which is the quiet path to data loss. Making both literals in the schema means the check costs nothing to write and cannot be forgotten — a migration step later has an unambiguous thing to switch on. _[earned 2026-08-11, first applied to the file-tree format]_

### T2: Recursive shapes are a TypeScript interface plus a `z.ZodType<T>` annotation, with `z.lazy` for the recursive field

Zod cannot infer a type that refers to itself, so the interface is written by hand and the schema is annotated with it. **Reason:** this keeps D6 intact — the schema is still the only definition, and the annotation is what forces the compiler to reject an interface that drifts from it. Writing the interface with `z.infer` instead does not work, and dropping the annotation lets the two drift apart silently. _[earned 2026-08-11, Zod 4.4.3]_

### T3: Paths inside documents are project-relative and forward-slashed, always

There is one conversion helper at the boundary and nothing else constructs a path string that reaches JSON. **Reason:** otherwise the same project serializes differently depending on which machine saved it, and every downstream comparison — diffs, tests, asset lookups — has to handle both. See `editor-kernel` G8. Absolute paths appear only in fields explicitly documented as machine-local, never in references. _[earned 2026-08-11]_

### T4: A format that is only ever served, never stored, still gets a schema

The file tree is computed per request and written to no file, and it is still defined as a Zod schema rather than a TypeScript type. **Reason:** the consumer is a separate process (the editor, or a human with a browser), so the shape is a contract across a boundary regardless of whether it lands on disk — and the round-trip test that guards it is only possible if a validator exists. _[earned 2026-08-11]_

### T5: A format with two producers gets one defaults factory, and they may still mint ids differently

The `.meta` format is written by the sidecar (for a file the human just dropped in) and by the sample-project generator (for content it authored). Both build their document from the same `defaultMeta` / `defaultImportSettings` functions in the schema module, so "what a fresh document looks like" is defined once. **They do not share id minting, and should not:** the sidecar mints random ids, the generator derives its ids from the file path so re-running it produces byte-identical output. **Reason:** the two differ in what they need from an id — uniqueness versus determinism — and agree on everything a *reader* can observe, because an id is opaque. Sharing the defaults prevents the drift that matters, one writer quietly emitting a different shape; forcing the id minting to be shared would break the generator's re-runnability for no reader's benefit. _[earned 2026-08-11]_

**There are four asset types and only three of them can be minted, which is the part to get right.** `texture`, `audio`, `font` — and **`other`**, for a file the editor has no import settings for at all. The distinction that matters: `other` is **never produced by the extension vocabulary** and so never by the sidecar, which is why `editor-kernel` D21's list of what the service will serve is correctly three and not four. It exists to be *read*, because a content generator has to mark a `.txt` it authored and a file that cannot carry its provenance inside itself needs a `.meta` to carry it (`editor-kernel` G4's marking rule).

So the defaults factory answers for four types and the sidecar reaches it with three. A regeneration that derives the type list from the extension map alone produces a schema that cannot parse a perfectly ordinary generated project — which is how the first parity drill found this. **Where a vocabulary has a member nothing mints, say so where the vocabulary is defined**, or the next reader will prune it as dead. _[earned 2026-08-12]_

### T6: Ids are validated as non-empty strings, never against a pattern

The convention (16 lowercase hex characters) lives in the function that mints them; the schema requires only a non-empty string. **Reason:** the schema's job is to catch drift between the format's own writers, and every writer goes through the minting function anyway — so a pattern adds nothing there, and costs something real elsewhere. It would turn an id a human typed by hand into a parse failure, and in a format whose whole promise is "read, never regenerated" a parse failure means the editor refusing to open a file it also refuses to fix. _[earned 2026-08-11]_

### T7: A field that appears twice in one document gets a `refine` saying the two must agree

`AssetMeta` states its type at the top level, where a reader looks for it, and again inside `importSettings`, where it discriminates the union. One `.refine` asserts they match. **Reason:** the duplication earns its place — one occurrence is for reading, the other for narrowing — but duplication without a check is just two places to be wrong. The line costs nothing and turns "somebody edited one of them" from a bug that surfaces in a panel into a parse error at the boundary. _[earned 2026-08-11, Zod 4.4.3]_

### T8: A served answer *about* a document is a different format from the document

`GET /meta` does not return an `AssetMeta`. It returns a small envelope carrying the path asked about, a status, the meta when there is one, and — when there is a file that will not parse — the reason and the file's raw text. **Reason:** the two answers that are not a document (there isn't one; there is one and this editor cannot read it) are the answers a panel most needs to say something useful about, and they have nowhere to live in the document's own schema. Bolting them on by making every field optional would destroy the document schema as a contract. Keep the envelope thin: anything the consumer can derive from the shared vocabulary it already imports (what type an extension is, say) does not belong in it. _[earned 2026-08-11]_

### T18: "This is not one of my documents" and "this is one of mine and it is broken" are different answers, and a served envelope has to carry which

`GET /document` answers `unreadable` for three quite different files: one claiming a format the editor knows that will not parse, one claiming a format it has never heard of, and one that is not JSON at all. The sentences already differed (T13's whole point); the *envelope* did not, so a consumer could only tell them apart by matching on prose.

**It surfaced as a feature that never worked in a real project.** The rename fixup reads every `.json` before it moves anything and refuses if one cannot be read — which is right for a level somebody mangled by hand, and absurd for the placeholder data tables every project has, whose formats do not exist yet. Every rename was refused, naming a file that could not possibly have held a reference.

**The fix is one boolean on the envelope, and getting its definition right is the whole of it.** `foreign` is true only when the file is readable and has **positively said it is not one of this editor's documents** — an unknown `format`, or none. It is false for a file that will not parse *at all*, because a file that says nothing about itself might be one of the editor's own that somebody mangled, and guessing otherwise is the guess that loses work. Three cases, one flag, and the flag is computed where the registry is rather than by each consumer.

Two rules that generalise past this field:

- **A served answer's *status* enum should not be widened for one consumer.** Four other readers wanted "unreadable, and here is the sentence"; adding a fourth status would have made all of them handle a case they do not care about. A field they can ignore is the cheaper shape (T8's thin-envelope rule, pointed at a distinction rather than at data).
- **Name the flag for the fact, not for the policy.** It was first written as `recognised`, which reads as "I know this format" — and then has to be *true* for a file that is not JSON, which nobody would guess. `foreign` states what the file did, and the awkward case stops being awkward. If a name forces a comment explaining why an obvious case is backwards, the name is wrong. _[earned 2026-08-12]_

### T9: A format the editor rewrites is built from loose objects, so the document it holds *is* the document from disk

Every object in `AssetMetaSchema` is `z.looseObject`, at every level, so keys the schema does not model survive the parse.

**Reason:** the editor rewrites a `.meta` from the object it parsed, so anything the parse drops is quietly deleted out of a file a human wrote — and the round-trip test does not catch it, because that test only ever compares fields the schema knows about (F1). The obvious alternative is a merge step that puts unmodelled keys back at write time, and it fails for three reasons stacked on top of each other: it has to be remembered, it has to be right at every nesting depth, and it has to know which subtrees are *replaced* wholesale rather than merged (a discriminated union whose shape changed — flipping a slice from `grid` to `single` must not leave `frameWidth` behind as debris). Loose objects have none of those failure modes, because nothing is ever put back: it never left. The costs are a word per object and an inferred type that gains no index signature; both are nothing.

The rule generalises. **A format the editor reads and later rewrites is loose. A format the editor only ever produces — a served answer, a computed view — stays strict**, because there is no human authorship in it to protect and strictness there catches a producer emitting a field nobody declared. `MetaViewSchema` is strict for exactly that reason while the document it carries is not. _[earned 2026-08-11, Zod 4.4.3]_

### T21: A field added to a format anything has already written is **optional and never defaulted**, and its absence is the old meaning

`opacity` joined the scene format's sprite component (`editor-kernel` D34) as `z.number().finite().optional()`, with "absent means fully solid" written on the field rather than into a default.

**Reason: a parse-time default is a rewrite of every document that never mentioned the field.** The editor rewrites a document from the object it parsed (T9), so a default lands on disk the next time anything touches the file — every entity in every level gaining `"opacity": 1` to say what it already said. That is a whole-project diff expressing nothing, and it buries any real change made the same day. The same argument bans "helpfully" normalising an absent value anywhere between the parse and the write.

Two consequences worth stating, because both were nearly got wrong:

- **A reader supplies the fallback, at the point of use, once.** `opacityOf` in the entity layer answers 1 for a sprite that never mentioned it; nothing else in the kernel has an opinion.
- **Every writer that rewrites the component must spread it.** A control that assigns `{ texture }` wholesale silently deletes the new field — the editor's texture picker did exactly that until it was changed to spread. T9's rule is usually stated about *unmodelled* keys; this is the same rule about a key the schema does model and one writer forgot.

The general test for a new field: **would a document that has never heard of this field be rewritten by adding it?** If yes, it is optional. _[earned 2026-08-15, Zod 4.4.3]_

### T10: A schema lives with the layer that ships, not with the process that writes it

The `.meta` schema was written by the filesystem service and lived beside it, and moved to `runtime/formats/` the first time the game runtime had to read import settings. **Reason and the rule to apply going forward:** ask "which layers read this?" when the format is created, and put it in the one that ships — a development-only process importing a shipping module is fine, the reverse is not (`editor-kernel` D1/D20). A schema compiled by more than one TypeScript project may have relative imports, and they must be written **`./x.js`** — the extensionless spelling only satisfies the browser project (`editor-kernel` D20, corrected 2026-08-12). Both are a one-line decision at creation and a cross-cutting move afterwards. _[earned 2026-08-11]_

### T11: A format that describes a file carries when it changed, not only how big it is

`FileNode` carries `mtimeMs` alongside `size`. **Reason:** size does not identify a version — a re-export from an art tool very often produces the same byte count — so anything keyed on size alone keeps serving the old contents after the human's edit (`editor-kernel` G11). Two details: it is `z.number().nonnegative()` rather than `.int()`, because some filesystems report finer than a millisecond and rounding would make the schema disagree with `stat`; and it costs nothing on disk because this format is served rather than stored (T4). _[earned 2026-08-11]_

### T12: A format with an open extension point pairs it with a registry, and validates by *checking* rather than replacing

A scene's `components` is `z.record(z.string(), z.unknown())` — any type key, any shape. Beside it, in the same file, is a registry naming the component types the kernel understands and the schema for each. A known type is validated against its schema; an unknown type is carried through untouched and reported to the Inspector as something this editor has no controls for.

**Reason, and the cost, stated plainly because this is a real concession.** A genre layer will add components, and requiring a kernel schema change for each is the thing that makes a kernel ossify. The price is that **the document schema stops being the whole truth about a document** (`editor-kernel` D6): a scene can hold a component this kernel has never heard of, and `tsc` will not know. What replaces D6's guarantee is the registry — one object, in one file, typechecked, and the same list the validator reads — so "what does this editor understand?" still has exactly one answer. Judge the bargain again if the registry ever grows a second copy anywhere.

Three details that are easy to get wrong, in the order they bite:

1. **Known components are checked, never replaced.** Running each value back through its own schema and storing the result would strip any key the component schema does not model — which is precisely what T9's loose objects exist to prevent, reintroduced one level down. A `superRefine` that validates and discards its result is the right shape, however odd it looks.
2. **An unknown type is not an error and a known-but-malformed one is.** These are different sentences and different situations: the first is most likely a newer editor or a genre layer, the second is a bug in something. Collapsing them into "invalid" makes the editor wrong about the common case.
3. **The registry lives in the schema file itself**, with the entity that carries a component map, because the entity schema validates against it and a registry in its own module would force a circular import. (An earlier version of this said the file *may have no relative imports*; that turned out to be false — see T14 — but the conclusion holds for the circularity reason alone.)

_[earned 2026-08-11, Zod 4.4.3]_

### T22: A *game's* description of a component is a fourth kind of format — it buys an inspector and deliberately not validation

T12 left the kernel with an open component map and a registry of the four types it owns, and one consequence nobody had answered: a component a game invents is carried perfectly and cannot be *authored*, because there is nothing on screen for it. The answer is a document a **game** writes describing one of its own components — a type, a title, and a list of fields with their kinds and defaults — which the editor reads and draws controls from. `kernel2d.component`, one file per component, in a `components/` folder beside the levels.

**The decision worth carrying, and it is a refusal.** T12 states the bargain as "registering a type is what buys validation and an inspector for it". A description buys the second half and **must not** buy the first:

- It is authored by a *game*, in the game's own folder, and a game-authored file must never be able to stop a level opening. Registering described types for validation means a typo in one file makes every level carrying that component refuse to parse — a fault in one folder taking out the project, with the panel that would explain it unreachable behind the same failure.
- So a described component is still an *unknown* component to the schema: carried byte-for-byte, never checked. The reading is lenient at the point of use instead — a field reports "the file holds something I cannot show" rather than throwing, and the panel says so out loud beside a control showing the default.
- **Two standards in one feature, on purpose.** The *description* is refused loudly when it is wrong (bad kind, no key, two fields writing to one key): it is one file, read once, and its author is looking at it. The *data* is never refused: it is in every level, and its author is three tools away.

Three smaller things that generalise:

1. **A description is a view of a component, not its schema.** So a writer spreads and sets one key rather than replacing the object — a key the description does not mention is a key some system reads. Replacing is right for a type the *kernel* owns and wrong here, and the two look identical in the diff.
2. **What describes a component and what reads it at runtime are different files, and nothing can enforce that they agree.** The description is editor-facing; the system in `src/` narrows the same keys by hand. A test in the *game's* folder asserting the described keys are the keys the system reads is the only guard available, and it belongs there rather than in the kernel — the kernel cannot know the correspondence exists.
3. ~~**Field kinds start at one.** `number`, written as a discriminated union with a single member, because the component it was proved on is three numbers. A reference kind was specifically *not* added, and the reason is a format fact rather than effort: reference fix-up on rename follows `COMPONENT_REFERENCE_FIELDS`, which a description does not add to — so the field would work and the rename would silently not, which is worse than no field.~~ **Struck the same day by T23**: the kinds are six, and the reference objection was answered by the description *stating* where its references are (`describedReferencesOf`) and the fix-up asking it — the same shape as the kernel's own map, so a rename follows a described file. What survives of the point: a discriminated union on `kind` with every reader switching on it was the right first shape, because the second kind was a case rather than a refactor.

_[earned 2026-08-15]_

### T23: A description's field kinds are six, two of them are references, and two keys are fixed by name because the walks cannot read descriptions

`number`, `text`, `toggle`, `choice`, `asset`, `scene` — one member each of a `kind`-discriminated union in `component-schema.ts`, all loose (T9). What each holds in a level, and the four decisions that were not obvious:

1. **`asset` is the D5 pair `{ id, path }` and `scene` is a bare path — because that is what the kernel's own references to each already are.** A texture or a sound has a `.meta` and therefore an id to witness with; a level is a JSON document with no `.meta`, so its one name is its path (T15, T20). Pairing a scene field with an id nobody can check would be T15's dead field.
2. **"Nothing chosen" is `null`, never an absent key.** `Add` writes every described key, and a game's system can then tell "no door yet" from "a level written before the field existed" — T15's rule for the project file, applied to a component. A `choice` default must be one of its options and the options' values must be unique; the description is refused otherwise.
3. **A `scene` field's key must be `scene`, and an `asset` field restricted to textures must be keyed `texture`** — refused by name otherwise. The reason is not taste: `sceneRefsOf` (which levels an export ships, T20) and `textureRefsOf` (which art a level preloads, T19) walk for *those key names* and know nothing about descriptions — one of them runs inside an exported game and could not read one if it wanted to. A door described as `{kind:'scene', key:'target'}` would author perfectly and be left out of the export; a texture called `icon` would be picked in the panel and not loaded when the level runs. **A description that works in the panel and fails out of sight is wrong, and the format's promise (T22) is that its own mistakes are refused loudly.** Cost accepted: one scene and one texture field per described component. An audio field, or one open to any file, can be called anything, because nothing walks for it.
4. **A kind the editor does not know is kept, not refused** — one fallback union member whose `kind` refines to "not a known kind", so a *malformed* known kind (a `number` with no `default`) is still refused with its own message rather than swallowed as unknown. The Inspector shows the field read-only; `Add` writes nothing for it. This is the level format's posture towards an unknown component type, one level in, and it is what lets a description written for a newer editor open in an older one.

And the reader: `readField` returns *what to show*, *whether it is really the file's*, and now **`held` — the raw thing in the file** — because the panel's answer to a value of the wrong kind changed from "show the default in a live control" to "show the file's own value with no control" (`editor-ui` U47, amended). Where a described component's references are is `describedReferencesOf(description)`, a format function the editor's rename fix-up asks — never a walk over `{id, path}`-shaped objects, which would be the editor guessing at what a game's data means. _[earned 2026-08-15]_

### T13: One list, and everything else derived from it

The service's document registry is `{ [format]: schema }` and the union type is `z.infer` over its values. Adding a format is one line, and the union, the "is this a format we know?" lookup and the validator that guards a write all follow from it.

**Reason:** the tempting alternative is a map for lookup plus a hand-written union type, which are the same list written twice and drift the first time somebody adds to one. The reason it cannot simply *be* a discriminated union is worth knowing: a union answers "did this parse", and the editor needs "is this a format I know" and "is this document malformed" to be different answers with different sentences (`editor-kernel` D22). Keying the map on the `format` literal every document already carries is what makes both available from one structure. _[earned 2026-08-11, Zod 4.4.3]_

### T14: One format may import another's vocabulary — and the constraint that said otherwise was never measured

The prefab format spent one session inside `scene-schema.ts` because a prefab holds a component map, the map is validated against the registry, the registry lives in the scene's file, and a shared-compiled module was recorded as unable to have relative imports of its own. Every link in that chain was true except the last one, which had been carried forward from an older entry and never tested.

**It is false.** `./scene-schema.js` — the `.js` spelling — resolves under bundler resolution, under NodeNext, and in Vite at runtime. One line, five minutes to check, and the prefab now has its own file.

**The rule that replaces it.** Two formats live in one file only when one *cannot* be expressed without the other's runtime values, and even then check that a one-way import would not do. What makes `prefab-schema.ts` → `scene-schema.ts` work is that the arrow runs one way: the prefab imports the registry, the scene never imports the prefab, and the one thing pointing back — an entity's `prefab` component — sits with the registry where components belong. A `Prefab` referenced from the scene's side would have to be a **type-only** import, which is erased and therefore not a runtime cycle; a value import both ways is the thing to refuse.

**The lesson that outlives the detail.** A constraint recorded without a measurement will shape designs for as long as nobody re-checks it, and it never announces itself — the code it produced compiles and passes. Anything in these skills phrased as "X is impossible" and not marked as measured is a candidate for a gardening pass to re-test. Cheap to check, and the cost of believing it is a file that quietly grows past what it should own. _[earned 2026-08-12, TypeScript 5.9.3, Vite 8]_

### T15: A reference with nothing to witness it is a path alone, said out loud — not a path plus an id that means nothing

`project.json` names the level a game starts on, and it names it as a plain project-relative path. Every other reference in the kernel carries a path *and* a stable id, because the path resolves it and the id notices when the file at that path is no longer the one it was written against (`editor-kernel` D5). This one cannot: a level has never carried an id, and only things pointed at from *inside* a level ever needed one (D24).

Three options, and the reason the middle one wins:

- **Path plus an id there is nothing to compare against.** Two fields, one of them permanently unread. An unread field is a field that is quietly wrong, and the next reader assumes it is checked because every other id is.
- **Path alone, with the schema saying why.** One field that works, and a comment naming the condition under which it should change — "if levels gain ids for another reason, this is the first place that should start carrying one".
- **Give levels an id so the reference can look like the others.** A format change to every level ever saved and to every writer of the format, bought for a warning nothing else in the kernel needs yet.

**Reason, generalised:** consistency between references is worth having, and it is worth *less* than every field being load-bearing. When a new reference cannot honour a convention, write down which half it is missing and why, in the schema, next to the field. That comment is the thing that stops a later session either trusting a dead field or "fixing" the inconsistency at the cost of a migration.

A second, smaller decision in the same file, for the same instinct: **"not chosen yet" is `nullable`, never optional.** An absent key and a key set to `null` would be two spellings of one state, and the editor would have to pick one to write while every reader handled both. _[earned 2026-08-12, Zod 4.4.3]_

### T16: A format owns the function that turns it into the bytes on disk, and the round-trip test goes through that function

Every schema module exports a `serialize…` beside the schema itself — `serializeMeta`, `serializeScene`, `serializePrefab`, `serializeProject` — and it is the only thing anywhere that renders a document as text. The round-trip test is written through it rather than through `JSON.stringify`, so what is proven is that **the text actually written to disk** reads back identical, not that some equivalent text would.

**Reason:** `CLAUDE.md` forbids a second serialization path, and this is the mechanism that makes the ban structural instead of a rule everyone has to remember. Indentation, key order and the trailing newline are all decisions a format makes once; a writer reaching for `JSON.stringify` directly makes them again, differently, and the result is a file that differs from the editor's own output in whitespace only. That costs nothing at parse time and everything in a diff — a folder where re-saving a level rewrites every line is a folder whose history is unreadable, and D3's whole case for text is git-diffability.

Two things that follow, both of which the first parity drill got wrong by not knowing this existed:

1. **The round trip is `parse(serialize(x))`, not `parse(JSON.parse(JSON.stringify(x)))`.** The second form is a weaker test that passes for a kernel whose writers disagree about whitespace, because whitespace is exactly what it throws away before comparing.
2. **A generated document is serialized the same way as an authored one.** That is what lets the sample generator promise byte-identical output on every run (G4) without owning a second opinion about formatting.

_[earned 2026-08-12, recorded after the first parity drill regenerated four formats without one]_

### T19: A component field named `texture` names a texture to load, wherever it sits — the field name is the whole contract

`textureRefsOf` (in the scene schema, beside `assetRefsOf`) walks an entity's component map at any depth — through unknown component types, through arrays — and collects every `AssetRef` sitting under a key called `texture`. Both texture-collection sites use it and only it: the runtime loader gathering what a level draws (`load-scene.ts`) and the editor's editing view (`scene-assets.tsx`), which must want the same set or the two pictures diverge (`editor-kernel` D2).

**Reason:** a game's systems may create entities *while a level runs* — projectiles, wave-spawned monsters — and nothing can fetch mid-run, so the art a spawned entity wears has to be loaded with the level. But the components that declare that art are the game's own inventions, which the kernel deliberately has no schema for. Walking for one agreed field name lets authored content say "this is drawable art" without the kernel learning a single genre word — the tower-defense game's `tower.projectile.texture` is loaded by a kernel that has no idea what a tower is. The convention was already universal before it was a rule: every texture reference the kernel itself writes sits under a key called `texture`.

Anything under a `texture` key that is not an `AssetRef` is skipped, not reported, on the same grounds as `assetRefsOf`: the walk answers "what art does this name", and a malformed reference names none. Worth knowing the boundary: `COMPONENT_REFERENCE_FIELDS` (rename fixups) does **not** walk this deep — a texture moved from inside the editor still fixes up sprite references but not ones nested in game components, which the id witness then reports at load instead. Generalising the fixup is a decision for the day it is somebody's real problem. _[earned 2026-08-14, first demanded by tower-defense projectiles]_ **Half-answered 2026-08-15 (T23):** a `texture` a *described* component holds at its top level is fixed up, because the description says it is a reference; the nested case is still open, and still reported by the witness rather than repaired.

### T20: A component field named `scene` names a scene the game can travel to — the mirror of T19, minus the id

`sceneRefsOf` (beside `textureRefsOf` in the scene schema) walks an entity's components at any depth and collects every non-empty string under a key called `scene`. Its consumer is the export closure (`editor-kernel` D13, amended): a game whose content holds doors — a level-select banner's `portal.scene`, a Ground's `home.scene` — ships every place those doors can reach, transitively, without the kernel learning what a portal is.

**Reason:** the door seam (`editor-kernel` D30) made travel a thing authored *content* can express, and the moment content can name a scene, tooling that answers "what does this game reach" must read the same naming or ship broken folders. One agreed field name is the whole contract, exactly as T19 — and it was, again, already the convention before it was a rule: the kernel's own story carrier and every game component naming a scene had used the key `scene` unprompted.

**Why a bare string where T19 collects an `AssetRef` pair:** a scene is a JSON document, so it has no `.meta` to hold an id, and its stable name *is* its project-relative path — the same name it wears in `project.json`, in the loader, and on the host's disk. There is no second naming to pair and therefore no witness to check; a wrong path is refused by whoever loads it, by name. A malformed or empty value names nothing and is skipped, on T19's grounds. _[earned 2026-08-14, demanded by tower-defense's level select]_

### T17: The `format` literal is data, so it belongs in the skills where a function name does not

The literals are `kernel2d.asset-meta`, `kernel2d.scene`, `kernel2d.prefab`, `kernel2d.project`, `kernel2d.component`, and the served-answer formats spell themselves the same way (`kernel2d.document-view`). Every document on disk carries one (T1), and the service's registry is keyed on them (T13).

**Reason, and the line this draws:** the first parity drill established that the skills carry decisions and reasons but not identifiers, and that this is correct — a regenerating session gets function names from the test suite, which ships alongside (`editor-kernel` D12). A `format` literal is not that kind of name. **It is a value written into every file a human has ever saved**, so a regenerated kernel that spells it differently compiles, passes a suite that was regenerated with it, and cannot open a single existing project. The test for whether an identifier belongs in a skill is therefore not "is it a name" but **"is it on disk in somebody's folder?"** _[earned 2026-08-12]_

### T24: A description can refuse to be handed out, because it can say what a component *holds* and never what it *belongs on*

`addable: false` in a description, absent meaning yes. It stops the editor offering the component to an entity that is not already one: the section is drawn only where the entity carries one or inherits one from its prefab, and nowhere else at all.

**The hole it patches is structural, not cosmetic, and it only shows up at the second described component.** With one description in a project the Add button is a feature. With two, the platformer's `walker` and `turtle` each grew an Add button on **all 249 entities of its level** — on clouds, on bricks, on the coin counter — every one of them offering to write a component no system in the game would ever read. And there is no way for the editor to do better on its own: nothing in a level marks an entity as an enemy, because *being* an enemy is carrying the `walker`, which is exactly the thing the button offers. An entity's own components are the only evidence available, so the only honest rule is one the description states about itself.

Three boundaries, each of which was the tempting wrong answer:

1. **A placement that inherits one is still offered Add.** It plainly is one of these already, and Add on an inherited component is how a single placement is given its own values and detached from the prefab — the one workflow this would otherwise have destroyed.
2. **Nothing is hidden that is really in the file.** A component sitting where it makes no sense still shows its fields and its Remove. Suppressing it would make the panel lie about a level to keep a rule tidy, which `editor-ui` U10 forbids; this key governs what may be *added*, never what is shown of what exists.
3. **No section at all, not a disabled button and not a sentence explaining the absence.** A greyed-out "Add walker" on a cloud is the same noise one indirection later.

Cheap because it is a *view* decision on a format that was already declared a view of a component rather than its schema (T22): one optional boolean, one exported reader holding the absent-means-yes default, one early return in the panel. Nothing about the data changed, so every description written before it keeps working unchanged. _[earned 2026-09-01, the platformer's two enemies]_

### T25: An entity's `parent` is an id alone, optional and absent — the first reference in a format that points inside its own file

`parent?: string` on the scene's `Entity` (`editor-kernel` D37, 2026-09-02): the id of another entity in the same list. Three things about its shape, each decided against a tempting alternative:

1. **An id alone, not T15's id-and-path pair.** The pair exists because a *file* can move and be renamed (D5); an entity in the same document has no path, and its id is already the name nothing changes. Carrying a name beside it would have been a second copy of a string the Outliner renames freely.
2. **Optional and absent, never `null` — T21 exactly.** The kernel's own constitution had written "defaulting to null" ahead of the feature; T21's test decided it: a `null` on every entity is a whole-project diff the day the editor next saves. The round-trip test therefore asserts two things, not one — the nested scene reads back identical, *and* `serializeScene` of a scene with nothing nested contains no `"parent"` anywhere.
3. **A parent that cannot be followed is not a parse failure.** The schema accepts an id that names nothing and a chain that loops; the loader reports both (`parent-missing`, `parent-cycle`) and places the entity by its own numbers. Refusing would answer a stray character with a level that will not open — D34's argument for opacity, one field over.

**The gotcha the report kinds taught:** they were the first members of `LoadProblem` with no `path`. Two helpers had quietly assumed every problem names a file — `describeLoadProblem` read `nameOf(problem.path)` *before* its switch, and the sort keyed on `path` — and both would have thrown at runtime with every type check green. A union gaining a member of a different shape needs its consumers re-read for what they assume of the *old* members, not only its switches extended. _[earned 2026-09-02]_

### T26: A prefab's `children` and a placement's `parts` are both optional-absent, and the id that joins them is not a format id

`runtime/formats/prefab-schema.ts` (2026-09-02, `editor-kernel` D25 amended). A prefab may carry `children: PrefabPart[]` — each `{ id, name, transform, parent?, components }`, the transform an offset from what the part rides — and a placement's `prefab` component may carry `parts: { [partId]: { [type]: value } }`. Three things decided:

1. **Both fields are optional and absent, never empty (T21).** A prefab that is one entity never mentions `children`; a placement that takes the prefab as it is never mentions `parts`. The editor's Remove-part deletes the list when it empties it, and the one override writer deletes an emptied record and then an emptied map — so both round-trip tests assert the *absence* of the key in `serializePrefab` / `serializeScene` output, not just equality after a parse.
2. **A part's id is stable within its prefab, and a drawn part's id is derived from it** — `<placement id>:<part id>`. The derived id is never written and no schema checks it; it exists in resolved lists only. A format id stays opaque (T6); this one carries meaning on purpose and is documented as resolution's contract, not the format's.
3. **The refusal has to follow the shape.** "A prefab may not contain a prefab" was one check on `components.prefab`; a `children` list and a `parts` map are two more places a `prefab` key could appear, and each has its own refusal. A format that gains a nested carrier must re-ask every invariant it holds on the outer one.

The override stores live *inside the reference they qualify* rather than beside it, so `copyEntity` carries them and reference rewriting never sees them. _[earned 2026-09-02]_

## Gotchas

### F2: A schema whose optional fields are `?: T` fails to typecheck under `exactOptionalPropertyTypes`

Zod infers an optional field as `T | undefined`, and with `exactOptionalPropertyTypes: true` a hand-written interface declaring `field?: T` is not assignable from it — the error names the *schema* line, not the interface, so it reads like a Zod problem. **Fix/policy:** write optional fields in the interface as `field?: T | undefined`. Same for any options object a strict project passes around with possibly-undefined values. _[earned 2026-08-11, Zod 4.4.3, TypeScript 5.9.3]_

### F1: A round-trip test proves the schema and the writer agree about what is kept, not that nothing was dropped

Zod strips keys the schema does not declare, so `parse(JSON.parse(JSON.stringify(x)))` will happily return a document with a writer's extra field removed, and the test still passes if the comparison is made against the stripped value. **Fix/policy:** compare the round-tripped result against the **original** object, not against a re-parse of itself — that is the comparison that fails when a writer emits a field the schema does not know about. _[earned 2026-08-11, Zod 4.4.3]_

**Sharpened once the editor started writing files.** The comparison above is necessary and not sufficient: it proves the schema and the writer agree, and says nothing about what happens to a key *a human* added that no writer knows about. For any format the editor rewrites, the test that matters is the one that hand-edits a parsed document with a key the schema has never heard of — at the top level *and* nested inside — and asserts it is still there after a write, a read and a write. That test is what fails the day somebody makes a loose schema strict again for tidiness (T9). _[earned 2026-08-11]_

## Contracts

- `kernel-2d/sidecar/tree-schema.ts` — the file-tree format: `ProjectTree`, `DirectoryNode`, `FileNode`, and the format/version literals. The first format in the kernel, and the worked example of T1–T4 and T11.
- `kernel-2d/tests/sidecar/tree-schema.test.ts` — the round-trip test every subsequent format copies.
- `kernel-2d/runtime/formats/meta-schema.ts` — the `.meta` format, and the worked example of T5–T7, T9 and T10: the discriminated union on `type`, the defaults factory every writer shares, the extension→type vocabulary, and why it sits in the layer that ships.
- `kernel-2d/sidecar/meta-view-schema.ts` — T8: the answer *about* a `.meta`, as distinct from the `.meta` itself.
- `kernel-2d/runtime/formats/scene-schema.ts` — the scene format, and the worked example of T12: the flat ordered entity list, the transform as a field rather than a component, the open component map, the registry beside it, and `COMPONENT_REFERENCE_FIELDS` (where a reference lives, as a fact about the format).
- `kernel-2d/runtime/formats/prefab-schema.ts` — the prefab format and `resolveEntity`, in their own file since T14: imports the scene's registry and is never imported back, which is the whole of why these are two files.
- `kernel-2d/sidecar/file-change-schema.ts` — T4 for an answer about *what was done* rather than about what is there: what a move or a delete reports, and why it has no spelling for a refusal.
- `kernel-2d/runtime/formats/project-schema.ts` — T15: the game's own settings, one field that works, and the reference that is a path alone with the reason written beside it.
- `kernel-2d/tests/runtime/project-schema.test.ts` — the round trip, the hand-added key that survives it, and the four refusals.
- `kernel-2d/scripts/export/manifest-schema.ts` — a format that describes a *folder* rather than a document: what an export wrote, addressed to the next export. The worked example of T4 for build output.
- `kernel-2d/runtime/formats/component-schema.ts` — T22: a game's description of one of its own components, the paragraph on why describing one buys an inspector and not validation, and the lenient field reader that keeps a panel honest without letting a level fail.
- `kernel-2d/tests/runtime/component-schema.test.ts` — the round trip, the refusals a description gets and the data does not, and the drift guard asserting a described type is still *not* in the kernel's own registry. Since T23: every kind read, refused and defaulted, and the unknown kind kept.
- `kernel-2d/tests/fixtures/door-description.ts` — T23: one description using every field kind and one that does not exist, shared by the schema tests, the reference fix-up tests and the browser suite (which writes it into the test project's `components/`).
- `kernel-2d/sidecar/document-view-schema.ts` — T13: the one list of document formats, and the answer *about* a document. Four formats in it now, and adding the fourth cost the same one line the second did.
- `kernel-2d/tests/runtime/scene-schema.test.ts` — the round trip, and the unknown component that survives it.
- `kernel-2d/tests/sidecar/document-endpoint.test.ts` — F1 sharpened, through the real service: a key added by hand surviving a write, at the top level and nested.
- `kernel-2d/tests/sidecar/meta-schema.test.ts` — the round trip plus the rejections that make the schema a contract rather than a suggestion.
