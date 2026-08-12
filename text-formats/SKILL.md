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

## Gotchas

### F2: A schema whose optional fields are `?: T` fails to typecheck under `exactOptionalPropertyTypes`

Zod infers an optional field as `T | undefined`, and with `exactOptionalPropertyTypes: true` a hand-written interface declaring `field?: T` is not assignable from it — the error names the *schema* line, not the interface, so it reads like a Zod problem. **Fix/policy:** write optional fields in the interface as `field?: T | undefined`. Same for any options object a strict project passes around with possibly-undefined values. _[earned 2026-08-11, Zod 4.4.3, TypeScript 5.9.3]_

### F1: A round-trip test proves the schema and the writer agree about what is kept, not that nothing was dropped

Zod strips keys the schema does not declare, so `parse(JSON.parse(JSON.stringify(x)))` will happily return a document with a writer's extra field removed, and the test still passes if the comparison is made against the stripped value. **Fix/policy:** compare the round-tripped result against the **original** object, not against a re-parse of itself — that is the comparison that fails when a writer emits a field the schema does not know about. _[earned 2026-08-11, Zod 4.4.3]_

## Contracts

- `kernel-2d/sidecar/tree-schema.ts` — the file-tree format: `ProjectTree`, `DirectoryNode`, `FileNode`, and the format/version literals. The first format in the kernel, and the worked example of T1–T4.
- `kernel-2d/tests/sidecar/tree-schema.test.ts` — the round-trip test every subsequent format copies.
- `kernel-2d/sidecar/meta-schema.ts` — the `.meta` format, and the worked example of T5–T7: the discriminated union on `type`, the defaults factory both writers share, and the extension→type vocabulary.
- `kernel-2d/sidecar/meta-view-schema.ts` — T8: the answer *about* a `.meta`, as distinct from the `.meta` itself.
- `kernel-2d/tests/sidecar/meta-schema.test.ts` — the round trip plus the rejections that make the schema a contract rather than a suggestion.
