---
description: Scaffold a new KiCad 10 project with the standard kicli layout — blank schematic, sym-lib-table, fp-lib-table, libs/ tree, and .kicli.toml. Usage — /kicli:new <project-name> [directory].
argument-hint: <project-name> [directory]
allowed-tools: Bash(kicli new:*), Bash(ls:*)
---

# kicli new project

Name: `$0`
Directory (optional): `$1` — defaults to a new folder named `$0`

## Steps

1. Run the appropriate form:
   - If `$1` is empty: `kicli new "$0"`
   - If `$1` is provided: `kicli new "$0" "$1"`
2. List the generated files (`ls -la $0` or the chosen dir) so the user sees the layout:
   - `<name>.kicad_pro` — KiCad 10 project file
   - `<name>.kicad_sch` — blank root schematic with UUID
   - `sym-lib-table` / `fp-lib-table` — pointing at the local `libs/`
   - `libs/symbols/`, `libs/footprints/`, `libs/3dmodels/` — empty library tree
   - `.kicli.toml` — project-level kicli config
3. Tell the user how to open it: `kicad $0/$0.kicad_pro`.
4. Suggest the next natural step: `/kicli:import <vendor.zip>` to pull in parts, or begin placing components in KiCad directly.

## Notes

- The name must not contain `/`, `\`, or `:`. kicli will reject these.
- If the directory already exists, kicli layers files in (idempotent for missing files) — but pre-existing `<name>.kicad_sch` / `.kicad_pro` will **not** be overwritten. Warn the user before running in a non-empty project dir.
