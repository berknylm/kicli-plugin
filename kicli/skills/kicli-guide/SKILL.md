---
name: kicli-guide
description: Reference for the `kicli` CLI — pipe-friendly KiCad 10 toolkit. Triggers whenever the conversation involves a `.kicad_sch` / `.kicad_pro` file, schematic review, pin/net connectivity, JLCPCB part lookup, BOM generation, component library import, or KiCad project scaffolding. Use this skill before writing ad-hoc bash — kicli has dedicated commands for every listed task.
user-invocable: false
allowed-tools: Bash(kicli:*)
---

# kicli — KiCad 10 agent toolkit

`kicli` is a pipe-friendly CLI that exposes KiCad 10 schematic data and JLCPCB part lookup in a grep/awk/cut-friendly form. **Never parse `.kicad_sch` by hand — use the kicli commands below.**

All output is plain text, tab- or CSV-separated. Exit codes: `0=ok`, `1=error`, `2=not-found`, `3=io`, `4=parse`. `kicli --json` is available on most read commands for structured output.

## Command map

| Task | Command |
|---|---|
| Inventory components | `kicli sch <file> list [--all]` → tab-separated `REF VALUE LIB FOOTPRINT PartNo` |
| Component detail | `kicli sch <file> info <REF>` |
| Pin+net connectivity | `kicli sch <file> view` → grep-friendly pin table |
| Edit one field | `kicli sch <file> set <REF> <FIELD> <VALUE>` |
| Bulk edit with filter | `kicli sch <dir> set-all <VALUE> <FIELD> <NEW> [--footprint <glob>] [--dry-run]` |
| Export | `kicli sch <file> export pdf\|svg\|netlist\|bom [-o FILE]` |
| ERC check | `kicli sch <file> erc [-o FILE]` |
| JLCPCB part detail | `kicli jlcpcb part <LCSC_ID>` → brand, stock, price, datasheet URL |
| JLCPCB search | `kicli jlcpcb search <query> [-n N] [--basic\|--extended] [--in-stock] [--package PKG]` |
| Project BOM | `kicli jlcpcb bom <file\|dir> [-o CSV]` (merges all `.kicad_sch`) |
| Import vendor ZIP | `kicli import <file.zip> [-l LIB] [--project DIR]` (SnapEDA, Ultra Librarian, CSE) |
| List imported libs | `kicli import --list` |
| Scaffold project | `kicli new <name>` |

## Key conventions

- **Pin type abbreviations in `view`:** `in, out, inout, pass, pwrin, pwrout, tri, oc, oe, free`
- **Net symbols in `view`:** `~` = floating (no wire), `NC` = intentional no-connect
- **`set-all` on a directory** applies to every `.kicad_sch` it contains
- **`--footprint <glob>`** prevents wrong-package part assignment in bulk edits
- **`--dry-run`** previews changes without writing — use this before any bulk edit
- **JLCPCB part tiers:** `basic` (no extra assembly fee) is preferred; `extended` costs more
- **LCSC IDs:** `C<digits>`, stored in the `LCSC` property (displays as `PartNo` in `list`)

## Schematic review workflow

```
1. kicli sch board.kicad_sch list           # inventory + PartNo status
2. kicli sch board.kicad_sch view           # full pin+net connectivity
3. view | grep '→ ~'                        # floating pins (ERC candidates)
4. view | grep '^U1:' | grep 'pwrin'        # power pins of IC U1
5. kicli jlcpcb part <LCSC_ID>              # fetch datasheet URL
6. Read datasheet, compare against view:
   - every VDD pin has a bypass cap nearby
   - pull-ups/downs match recommended values
   - crystal load caps match spec
   - unused inputs tied appropriately
```

## BOM preparation workflow

```
1. kicli jlcpcb bom project/                        # identifies empty PartNo rows
2. For each missing row:
   a. kicli jlcpcb search "<value> <package>" --basic --in-stock
   b. Pick with circuit context in mind (voltage, temp, tolerance)
   c. kicli jlcpcb part <LCSC_ID>                   # verify before assigning
   d. kicli sch proj/ set-all "<value>" LCSC <id> --footprint "*<pkg>*" --dry-run
   e. Review diff, then remove --dry-run to apply
3. kicli jlcpcb bom project/                        # confirm all rows filled
4. kicli jlcpcb bom project/ -o bom.csv             # export for JLCPCB upload
```

## Vendor library import workflow

```
1. kicli import ~/Downloads/<part>.zip [-l <library-name>]
   - Vendor prefixes (ul_, LIB_) auto-stripped
   - Symbol, footprint, 3D model imported
   - libs/symbols/, libs/footprints/, libs/3dmodels/ populated
   - sym-lib-table + fp-lib-table updated automatically
2. Component available as <library>:<component> inside KiCad
3. kicli import --list                              # verify registration
```

## Common pipe recipes

```bash
kicli sch board.kicad_sch list | grep '^U'                     # all ICs
kicli sch board.kicad_sch list | awk '$5 == ""'                # missing PartNo
kicli sch board.kicad_sch list | cut -c1 | sort | uniq -c      # prefix histogram
kicli sch board.kicad_sch view | grep '→ ~'                    # floating pins
kicli sch board.kicad_sch view | grep '→ VDDA_3V3'             # everything on a net
diff <(kicli sch old.kicad_sch list) <(kicli sch new.kicad_sch list)
kicli jlcpcb part C89418 | grep Datasheet | cut -d' ' -f2      # datasheet URL only
kicli jlcpcb bom project/ | grep ',$' && echo "INCOMPLETE" || echo "READY"
```

## Design philosophy — important

kicli is a **data layer**, not a design engine. It reads, displays, and edits — it never automates component selection because that requires circuit context (voltage, current, temperature, physical constraints) that only you + the datasheet know. Always read the datasheet before assigning a part.

## Binary location

kicli must be in `$PATH`. If missing, install from [github.com/berknylm/kicli/releases](https://github.com/berknylm/kicli/releases) (Linux, macOS, Windows x86_64). KiCad 10 is required — `kicli config kicad-path <path>` if auto-detection fails.
