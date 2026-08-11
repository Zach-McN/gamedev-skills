# Provenance

Source: `npm pack phaser@4.1.0` (official npm tarball), vendored 2026-08-11.
4.1.0 is the only release on the 4.1.x line (npm `latest` was 4.2.1 at vendoring time).

Contents, with original paths inside the tarball:

| File here | Original path in `package/` |
|---|---|
| `types/phaser.d.ts` | `types/phaser.d.ts` |
| `types/index.d.ts` | `types/index.d.ts` |
| `types/matter.d.ts` | `types/matter.d.ts` |
| `CHANGELOG.md` | `CHANGELOG.md` |
| `changelog/CHANGELOG-v4.0.0.md` | `changelog/v4/4.0/CHANGELOG-v4.0.0.md` |
| `changelog/CHANGELOG-v4.1.0.md` | `changelog/v4/4.1/CHANGELOG-v4.1.0.md` |
| `MIGRATION-GUIDE.md` | `changelog/v4/4.0/MIGRATION-GUIDE.md` |
| `v3-to-v4-migration-SKILL.md` | `skills/v3-to-v4-migration/SKILL.md` |

These files are read-only reference (see repo CLAUDE.md): never edit them; replace them wholesale when the pinned version changes, and update this file when you do.

Note: the phaser npm package also ships a full `skills/` directory (official Phaser 4 skills per topic) and `docs/` guides (rendering concepts, shaders, pixel art). Not vendored here to keep the footprint small — re-run `npm pack phaser@4.1.0` if a session needs them.
