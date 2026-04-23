# kicli plugin for Claude Code

AI-native KiCad 10 workflow inside Claude Code. Wraps the [`kicli`](https://github.com/berknylm/kicli) CLI so Claude can read schematics, resolve pin/net connectivity, look up JLCPCB parts, prepare BOMs, and import vendor component ZIPs — all through pipe-friendly commands.

## What it gives you

**Slash commands**

| Command | Action |
|---|---|
| `/kicli:review <sch\|dir>` | End-to-end schematic review: inventory → connectivity → floating pins → datasheet cross-check → ERC |
| `/kicli:bom <sch\|dir> [out.csv]` | Generate a JLCPCB-ready BOM, fill missing part numbers, export |
| `/kicli:search <query>` | JLCPCB part search with tier/stock/package filters |
| `/kicli:import <zip> [lib]` | Import vendor ZIP (SnapEDA, Ultra Librarian, CSE) into the project's `libs/` |
| `/kicli:new <name>` | Scaffold a new KiCad 10 project with the kicli layout |

**Passive skill** (`kicli-guide`) — loads a condensed CLI reference whenever the conversation involves a `.kicad_sch`, `.kicad_pro`, JLCPCB part, or BOM. Claude stops guessing at s-expression internals and runs the right `kicli` command instead.

**Session hook** — at session start, prints the installed kicli version or a friendly install hint if it's missing.

## Prerequisites

1. **kicli** ≥ v0.5.1 — [releases](https://github.com/berknylm/kicli/releases/latest). Download the appropriate archive (Linux, macOS x86_64/arm64, Windows x86_64), extract, put `kicli` somewhere in `$PATH`.
2. **KiCad 10** — [kicad.org/download](https://www.kicad.org/download/). kicli auto-detects it in the default install locations. If you installed it somewhere custom: `kicli config kicad-path /path/to/kicad-cli`.

## Installation

Inside Claude Code:

```
/plugin marketplace add berknylm/kicli-plugin
/plugin install kicli@kicli-marketplace
```

Or, for local development — add the cloned repo as a local marketplace:

```
/plugin marketplace add /path/to/kicli-plugin
/plugin install kicli@kicli-marketplace
```

Reload after editing the plugin without restarting Claude Code:

```
/reload-plugins
```

## Quick tour

```
> /kicli:review hardware/board.kicad_sch
→ runs list, view, greps floating pins, looks up each IC's datasheet,
  compares connectivity, reports blockers/warnings/ok

> /kicli:bom hardware/ out.csv
→ merges every .kicad_sch, flags empty PartNo rows,
  proposes JLCPCB candidates, bulk-assigns with --footprint filter,
  exports JLCPCB-compatible CSV

> /kicli:search "100nF 0402"
→ kicli jlcpcb search with --basic --in-stock, top 10,
  detail for top 3, comparison table
```

## How it relates to kicli

The plugin does **not** ship the kicli binary. kicli is a standalone C11/CMake CLI that runs outside Claude — the plugin just teaches Claude the command surface and workflows. You can use kicli from any shell; this plugin makes it first-class inside Claude Code.

- kicli repo + source: [github.com/berknylm/kicli](https://github.com/berknylm/kicli)
- kicli docs + recipes: [SKILLS.md](https://github.com/berknylm/kicli/blob/main/SKILLS.md) (embedded; runtime access via `kicli skills`)

## License

MIT — see [LICENSE](LICENSE).
