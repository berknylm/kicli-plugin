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

All **read** commands (`list`, `info`, `view`) accept either a single `.kicad_sch`
or a **project directory** — in dir mode they walk every sheet.

| Task | Command |
|---|---|
| Inventory components | `kicli sch <file\|dir> list [--all]` → tab-separated `REF\tVALUE\tLIB\tFOOTPRINT\tPartNo` (dir mode adds a 6th column `SHEET`; missing PartNo prints as `(unset)`) |
| Component detail | `kicli sch <file\|dir> info <REF> [--pins]` (`--pins` adds full pin table `NUM\tNAME\tTYPE\tNET`) |
| Pin+net connectivity | `kicli sch <file\|dir> view [-o FILE]` (dir mode resolves nets across sheets via the root `.kicad_sch`'s netlist — sheet-pin ↔ hierarchical-label bridges show under one net name) |
| Edit one field | `kicli sch <file> set <REF> <FIELD> <VALUE>` |
| Bulk edit with filter | `kicli sch <file\|dir> set-all <VALUE> <FIELD> <NEW> [--footprint <glob>] [--dry-run]` |
| Export | `kicli sch <file> export pdf\|svg\|netlist\|bom [-o FILE]` |
| ERC check | `kicli sch <file> erc [-o FILE\|-]` (`-o -` streams the report to stdout) |
| JLCPCB part detail | `kicli jlcpcb part <LCSC_ID>` → brand, stock, price, datasheet URL |
| JLCPCB search | `kicli jlcpcb search <query> [-n N] [--basic\|--extended] [--in-stock] [--package PKG]` |
| Project BOM | `kicli jlcpcb bom <file\|dir> [-o CSV]` (merges all `.kicad_sch`) |
| Import vendor ZIP | `kicli import <file.zip> [-l LIB] [--project DIR]` (SnapEDA, Ultra Librarian, CSE) |
| List imported libs | `kicli import --list` |
| Scaffold project | `kicli new <name>` |

## Key conventions

- **Pin type abbreviations in `view` / `info --pins`:** `in, out, inout, pass, pwrin, pwrout, tri, oc, oe, nc, -`
- **Net symbols in `view`:** `~` = floating (no wire), `NC` = intentional no-connect
- **`(unset)`** in the PartNo column means the `LCSC` property is missing — this is the row to fill for BOM prep
- **Dir mode for `list` / `view` / `info`** walks every `.kicad_sch` in the project; `view` additionally runs `kicad-cli` netlist on the root schematic so cross-sheet nets resolve consistently
- **`--footprint <glob>`** prevents wrong-package part assignment in bulk edits
- **`--dry-run`** previews changes without writing — use this before any bulk edit
- **JLCPCB part tiers:** `basic` (no extra assembly fee) is preferred; `extended` costs more
- **LCSC IDs:** `C<digits>`, stored in the `LCSC` property (displays as `PartNo` in `list`)

## Schematic review workflow

```
1. kicli sch project/ list                  # project-wide inventory + PartNo
                                            # (SHEET column shows each sheet)
2. kicli sch project/ view                  # project-wide pin+net view, with
                                            # nets resolved across sheet pins
                                            # ↔ hierarchical labels
3. view | grep '→ ~'                        # floating pins (ERC candidates)
4. view | grep '/U1:\|^U1:' | grep 'pwrin'  # power pins of U1 on every sheet
5. kicli sch project/ info U1 --pins        # full pin table (NUM NAME TYPE NET)
6. kicli jlcpcb part <LCSC_ID>              # fetch datasheet URL
7. Read datasheet, compare against view:
   - every VDD pin has a bypass cap nearby
   - pull-ups/downs match recommended values
   - crystal load caps match spec
   - unused inputs tied appropriately
8. kicli sch project/top.kicad_sch erc -o - # stream ERC report to stdout
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
# project-wide queries
kicli sch project/ list | grep '^U'                              # all ICs
kicli sch project/ list | awk -F'\t' '$5 == "(unset)"'           # missing PartNo
kicli sch project/ list | awk -F'\t' '{print $6}' | sort | uniq -c  # per-sheet count
kicli sch project/ view | grep '→ ~'                             # floating pins
kicli sch project/ view | grep '→ +3.3V'                         # every pin on a net
kicli sch project/ info U1 --pins                                # full pin dump

# single-file queries still work
kicli sch board.kicad_sch list | cut -c1 | sort | uniq -c       # prefix histogram
diff <(kicli sch old.kicad_sch list) <(kicli sch new.kicad_sch list)

# JLCPCB + BOM
kicli jlcpcb part C89418 | grep Datasheet | cut -d' ' -f2       # datasheet URL only
kicli jlcpcb bom project/ | grep ',$' && echo "INCOMPLETE" || echo "READY"

# ERC
kicli sch project/top.kicad_sch erc -o - | grep -E '^\[|violations'
```

## Design philosophy — important

kicli is a **data layer**, not a design engine. It reads, displays, and edits — it never automates component selection because that requires circuit context (voltage, current, temperature, physical constraints) that only you + the datasheet know. Always read the datasheet before assigning a part.

## Binary location

kicli must be in `$PATH`. If missing, install from [github.com/berknylm/kicli/releases](https://github.com/berknylm/kicli/releases) (Linux, macOS, Windows x86_64). KiCad 10 is required — `kicli config kicad-path <path>` if auto-detection fails.
