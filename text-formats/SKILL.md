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

### T6: Ids are validated as non-empty strings, never against a pattern

The convention (16 lowercase hex characters) lives in the function that mints them; the schema requires only a non-empty string. **Reason:** the schema's job is to catch drift between the format's own writers, and every writer goes through the minting function anyway — so a pattern adds nothing there, and costs something real elsewhere. It would turn an id a human typed by hand into a parse failure, and in a format whose whole promise is "read, never regenerated" a parse failure means the editor refusing to open a file it also refuses to fix. _[earned 2026-08-11]_

### T7: A field that appears twice in one document gets a `refine` saying the two must agree

`AssetMeta` states its type at the top level, where a reader looks for it, and again inside `importSettings`, where it discriminates the union. One `.refine` asserts they match. **Reason:** the duplication earns its place — one occurrence is for reading, the other for narrowing — but duplication without a check is just two places to be wrong. The line costs nothing and turns "somebody edited one of them" from a bug that surfaces in a panel into a parse error at the boundary. _[earned 2026-08-11, Zod 4.4.3]_

### T8: A served answer *about* a document is a different format from the document

`GET /meta` does not return an `AssetMeta`. It returns a small envelope carrying the path asked about, a status, the meta when there is one, and — when there is a file that will not parse — the reason and the file's raw text. **Reason:** the two answers that are not a document (there isn't one; there is one and this editor cannot read it) are the answers a panel most needs to say something useful about, and they have nowhere to live in the document's own schema. Bolting them on by making every field optional would destroy the document schema as a contract. Keep the envelope thin: anything the consumer can derive from the shared vocabulary it already imports (what type an extension is, say) does not belong in it. _[earned 2026-08-11]_

### T9: A format the editor rewrites is built from loose objects, so the document it holds *is* the document from disk

Every object in `AssetMetaSchema` is `z.looseObject`, at every level, so keys the schema does not model survive the parse.

**Reason:** the editor rewrites a `.meta` from the object it parsed, so anything the parse drops is quietly deleted out of a file a human wrote — and the round-trip test does not catch it, because that test only ever compares fields the schema knows about (F1). The obvious alternative is a merge step that puts unmodelled keys back at write time, and it fails for three reasons stacked on top of each other: it has to be remembered, it has to be right at every nesting depth, and it has to know which subtrees are *replaced* wholesale rather than merged (a discriminated union whose shape changed — flipping a slice from `grid` to `single` must not leave `frameWidth` behind as debris). Loose objects have none of those failure modes, because nothing is ever put back: it never left. The costs are a word per object and an inferred type that gains no index signature; both are nothing.

The rule generalises. **A format the editor reads and later rewrites is loose. A format the editor only ever produces — a served answer, a computed view — stays strict**, because there is no human authorship in it to protect and strictness there catches a producer emitting a field nobody declared. `MetaViewSchema` is strict for exactly that reason while the document it carries is not. _[earned 2026-08-11, Zod 4.4.3]_

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

### T13: One list, and everything else derived from it

The service's document registry is `{ [format]: schema }` and the union type is `z.infer` over its values. Adding a format is one line, and the union, the "is this a format we know?" lookup and the validator that guards a write all follow from it.

**Reason:** the tempting alternative is a map for lookup plus a hand-written union type, which are the same list written twice and drift the first time somebody adds to one. The reason it cannot simply *be* a discriminated union is worth knowing: a union answers "did this parse", and the editor needs "is this a format I know" and "is this document malformed" to be different answers with different sentences (`editor-kernel` D22). Keying the map on the `format` literal every document already carries is what makes both available from one structure. _[earned 2026-08-11, Zod 4.4.3]_

### T14: One format may import another's vocabulary — and the constraint that said otherwise was never measured

The prefab format spent one session inside `scene-schema.ts` because a prefab holds a component map, the map is validated against the registry, the registry lives in the scene's file, and a shared-compiled module was recorded as unable to have relative imports of its own. Every link in that chain was true except the last one, which had been carried forward from an older entry and never tested.

**It is false.** `./scene-schema.js` — the `.js` spelling — resolves under bundler resolution, under NodeNext, and in Vite at runtime. One line, five minutes to check, and the prefab now has its own file.

**The rule that replaces it.** Two formats live in one file only when one *cannot* be expressed without the other's runtime values, and even then check that a one-way import would not do. What makes `prefab-schema.ts` → `scene-schema.ts` work is that the arrow runs one way: the prefab imports the registry, the scene never imports the prefab, and the one thing pointing back — an entity's `prefab` component — sits with the registry where components belong. A `Prefab` referenced from the scene's side would have to be a **type-only** import, which is erased and therefore not a runtime cycle; a value import both ways is the thing to refuse.

**The lesson that outlives the detail.** A constraint recorded without a measurement will shape designs for as long as nobody re-checks it, and it never announces itself — the code it produced compiles and passes. Anything in these skills phrased as "X is impossible" and not marked as measured is a candidate for a gardening pass to re-test. Cheap to check, and the cost of believing it is a file that quietly grows past what it should own. _[earned 2026-08-12, TypeScript 5.9.3, Vite 8]_

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
- `kernel-2d/runtime/formats/scene-schema.ts` — the scene format *and* the prefab format, and the worked example of T12 and T14: the flat ordered entity list, the transform as a field rather than a component, the open component map, the registry beside it, and the reference between the two documents.
- `kernel-2d/sidecar/document-view-schema.ts` — T13: the one list of document formats, and the answer *about* a document.
- `kernel-2d/tests/runtime/scene-schema.test.ts` — the round trip, and the unknown component that survives it.
- `kernel-2d/tests/sidecar/document-endpoint.test.ts` — F1 sharpened, through the real service: a key added by hand surviving a write, at the top level and nested.
- `kernel-2d/tests/sidecar/meta-schema.test.ts` — the round trip plus the rejections that make the schema a contract rather than a suggestion.
