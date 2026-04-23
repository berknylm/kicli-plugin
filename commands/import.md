---
description: Import a vendor component ZIP (SnapEDA, Ultra Librarian, Component Search Engine) into the current KiCad project. Usage — /kicli:import <file.zip> [library-name].
argument-hint: <file.zip> [library-name]
allowed-tools: Bash(kicli import*), Bash(ls*), Read
---

# kicli vendor import

Target ZIP: `$0`
Library name override: `$1` (optional — defaults to the normalized filename stem)

## Steps

1. Verify the file exists and ends in `.zip`. If the user passed a bare part name, suggest the expected location (e.g. `~/Downloads/$0.zip`).
2. If a library name is given (`$1`), run:
   ```
   kicli import "$0" -l "$1"
   ```
   Otherwise:
   ```
   kicli import "$0"
   ```
3. kicli auto-detects the project root (ancestor with `*.kicad_pro`). If that fails it will say so — then re-run with `--project <dir>`.
4. After import, run `kicli import --list` and confirm the new library appears.
5. Summarize what was installed:
   - Symbol lib → `libs/symbols/<lib>.kicad_sym`
   - Footprint → `libs/footprints/<lib>.pretty/<component>.kicad_mod`
   - 3D model → `libs/3dmodels/<lib>/<component>.step`
   - Registered in `sym-lib-table` and `fp-lib-table`
6. Tell the user how to reference the new component in KiCad: `<library>:<component>`.

## Vendor cues

- **SnapEDA** — ZIP contents split across top-level files; kicli picks `.kicad_sym`, `.kicad_mod`, `.step` automatically.
- **Ultra Librarian** — filenames prefixed with `ul_` (auto-stripped). Has a `KiCad/` subdirectory which kicli prioritizes over generic alternatives.
- **Component Search Engine (CSE)** — `LIB_` prefix auto-stripped. Usually ships both KiCad-native and generic files; kicli prefers the KiCad variants.

## If something is missing

If kicli reports "no KiCad files found" or only legacy `.lib` symbols, tell the user — do not attempt to fix the ZIP. Ask them to re-download from a vendor that provides KiCad 10 format.
