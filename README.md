# kicli plugin for Claude Code

AI-native KiCad 10 workflow inside Claude Code. Wraps the
[`kicli`](https://github.com/berknylm/kicli) CLI so Claude can read
schematics, resolve pin/net connectivity, look up JLCPCB parts, prepare
BOMs, browse symbol / footprint catalogs, and import vendor component
ZIPs — all through pipe-friendly commands.

Works in **Claude Code Desktop** (macOS / Windows app), **Claude Code CLI**,
and any other Claude Code surface that supports plugins.

---

## Install

### Step 1 — Install the `kicli` binary (prerequisite, not bundled)

Download the correct archive for your platform from
[kicli releases](https://github.com/berknylm/kicli/releases/latest):

```bash
# macOS (Apple Silicon)
curl -L https://github.com/berknylm/kicli/releases/latest/download/kicli-macos-arm64.tar.gz | tar -xz
sudo mv kicli /usr/local/bin/

# macOS (Intel)
curl -L https://github.com/berknylm/kicli/releases/latest/download/kicli-macos-x86_64.tar.gz | tar -xz
sudo mv kicli /usr/local/bin/

# Linux
curl -L https://github.com/berknylm/kicli/releases/latest/download/kicli-linux-x86_64.tar.gz | tar -xz
sudo mv kicli /usr/local/bin/

# Windows — download kicli-windows-x86_64.zip, unzip, add the folder to PATH
```

Verify:

```bash
kicli --version
# → kicli 0.9.0  (or newer)
```

You also need **KiCad 10** installed — [kicad.org/download](https://www.kicad.org/download/).
kicli auto-detects it in the default install paths.

### Step 2 — Install the plugin

#### Inside Claude Desktop's "Claude Code" tab (recommended)

Open a Claude Code session in Claude Desktop and type **exactly**:

```
/plugin marketplace add berknylm/kicli-plugin
```

Hit Enter, wait for `✔ Successfully added marketplace`. Then:

```
/plugin install kicli@kicli-marketplace
```

Claude Code will say: `✔ Plugin "kicli" updated ... Restart to apply changes.`

**Quit Claude Desktop completely** (`Cmd+Q` on macOS, `Alt+F4` → quit on
Windows — closing the window isn't enough) and reopen it.

#### Or from a terminal

```bash
claude plugin marketplace add berknylm/kicli-plugin
claude plugin install kicli@kicli-marketplace
# Then fully quit + reopen Claude Desktop.
```

### Step 3 — Verify it's wired up

In a **new** Claude Code session (not a resumed one), type `/kicli` and hit Tab.
Autocomplete should offer five commands:

```
/kicli:bom       /kicli:import    /kicli:new       /kicli:review    /kicli:search
```

You should also see this line from the session-start hook at the very top
of the new chat:

```
SessionStart:resume hook success: kicli 0.9.0
```

(If the line says `kicli not found in PATH`, go back to Step 1.)

---

## Slash commands

| Command | Action |
|---|---|
| `/kicli:review <sch\|dir>` | End-to-end schematic review: inventory → pin/net view → floating pins → datasheet cross-check → ERC |
| `/kicli:bom <sch\|dir> [out.csv]` | JLCPCB-ready BOM; fill missing parts; export |
| `/kicli:search <query>` | JLCPCB part search with tier/stock/package filters |
| `/kicli:import <zip> [lib]` | Import SnapEDA / Ultra Librarian / CSE ZIP into the project's `libs/` |
| `/kicli:new <name>` | Scaffold a new KiCad 10 project |

**Passive skill** `kicli-guide` — auto-loads a condensed CLI reference
whenever the chat mentions a `.kicad_sch` / `.kicad_pro`, JLCPCB part,
BOM, or footprint. Claude stops guessing s-expression internals and runs
the right `kicli` command directly (including the catalog ones like
`kicli sym search` and `kicli fp search`).

---

## Quick tour

```
> /kicli:review hardware/

  → walks the project directory, runs list + view (cross-sheet netlist),
    greps floating pins, looks up each IC's datasheet via JLCPCB, cross-
    checks assignments with `jlcpcb check`, produces a blockers /
    warnings / ok report.

> /kicli:bom hardware/ out.csv

  → merges every .kicad_sch, flags empty PartNo rows, proposes candidates
    via `jlcpcb search --basic --in-stock`, bulk-assigns using
    `sch set-all --footprint <glob> --only-empty`, exports the JLCPCB
    upload CSV.

> /kicli:search "100nF 0402"

  → jlcpcb search --basic --in-stock --package 0402, top 10, detail for
    top 3, comparison table. Agent picks based on circuit context.

> I have an LM358 zip from SnapEDA, import it.
  → (kicli-guide skill fires) kicli import LM358.zip -l op_amps
```

---

## Troubleshooting

**`/kicli:*` commands don't autocomplete after install.**
Claude Desktop must be fully restarted (`Cmd+Q`, not just closing the
window). Commands are loaded at session start and cached. If you were
already inside a session during install, the plugin files are on disk but
the session's system prompt is stale. A `/reload-plugins` inside the
session may pick them up; a full quit + reopen always does.

**SessionStart hook prints `kicli not found in PATH`.**
You haven't installed the binary yet (Step 1) or the shell that Claude
Desktop uses can't see it. On macOS/Linux, put it in `/usr/local/bin/`.
On Windows, make sure the `kicli.exe` folder is in the system PATH
(user PATH isn't always picked up by GUI apps — use the machine-wide
PATH variable).

**`/plugin marketplace add berknylm/kicli-plugin` errors.**
Check network reachability to GitHub. Git is invoked under the hood;
corporate proxies / SSH key prompts can block the clone. Workaround:
use HTTPS explicitly:

```
/plugin marketplace add https://github.com/berknylm/kicli-plugin.git
```

**`/kicli:review` runs but `kicli` errors with "kicad-cli not found".**
KiCad 10 isn't installed, or kicli's auto-detection missed your install
path. Fix with:

```bash
kicli config kicad-path /path/to/kicad-cli
```

(On macOS the path is usually `/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli`.)

**Updating to a newer plugin version.**

```
/plugin marketplace update kicli-marketplace
/plugin update kicli@kicli-marketplace
```

Restart Claude Desktop to pick up the new commands / skill text.

**Uninstalling.**

```
/plugin uninstall kicli@kicli-marketplace
/plugin marketplace remove kicli-marketplace
```

---

## What does NOT ship in the plugin

The plugin is a thin teaching layer — **it does not bundle the kicli
binary**. kicli is a standalone C11/CMake CLI you install separately
(Step 1 above); the plugin just tells Claude how to drive it and what
the output shape is. You can use `kicli` from any shell, from CI, or
from a script regardless of whether this plugin is installed.

The plugin also does not ship KiCad itself. kicli needs `kicad-cli`
for schematic netlist resolution, ERC, export — install KiCad 10 from
[kicad.org/download](https://www.kicad.org/download/).

---

## Links

- kicli source + issues: [github.com/berknylm/kicli](https://github.com/berknylm/kicli)
- kicli CLI reference: [SKILLS.md](https://github.com/berknylm/kicli/blob/main/SKILLS.md)
  (also embedded in the binary — runtime access: `kicli skills`)
- Plugin changelog: [releases](https://github.com/berknylm/kicli-plugin/releases)

---

## License

MIT — see [LICENSE](LICENSE).
