---
name: editor-kernel
description: The architecture constitution for AI-built game editors — the core decisions, invariants, and contracts that every editor kernel must obey (document model, undo/redo, tool dispatch, persistence boundaries). Consult this skill whenever building or modifying the kernel of a game editor, deciding how editor state is structured, or resolving an architectural question about how the editor's pieces fit together.
---

# editor-kernel

The architecture constitution for the game-editor kernel. This skill holds the load-bearing decisions about how an editor is structured — the document model, how state mutates and undoes, how tools dispatch, and where the boundaries sit between editor, runtime, and persistence. It is the highest-authority skill in the library: when another skill conflicts with a decision recorded here, this one wins. Content is earned from real kernel sessions, never invented.

## Decisions

_None recorded yet. Filled via dual-write as the kernel is built._

## Gotchas

_None recorded yet. Filled via dual-write as the kernel is built._

## Contracts

_None recorded yet. Reference kernel-repo and vendored-doc file paths here; never paraphrase schemas as prose._
