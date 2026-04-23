---
description: End-to-end KiCad schematic review — inventory, connectivity, floating pins, datasheet cross-check. Usage — /kicli:review <sch-file-or-project-dir>.
argument-hint: <file.kicad_sch | project/>
allowed-tools: Bash(kicli:*), Bash(grep:*), Bash(awk:*), Bash(sort:*), Bash(cut:*), Read, WebFetch
---

# kicli schematic review

Target: `$ARGUMENTS`

Perform a thorough, grounded review of the schematic. **Do not guess — run the commands and report what kicli shows.**

## Pass 1 — Inventory & completeness

1. `kicli sch $ARGUMENTS list` — print the component table. If `$ARGUMENTS` is a directory, a 6th column SHEET is appended so you can see which sheet each symbol is on.
2. Flag components with `(unset)` in the PartNo column — those block BOM generation. Pipe: `kicli sch $ARGUMENTS list | awk -F'\t' '$5 == "(unset)"'`.
3. Histogram by reference prefix: `kicli sch $ARGUMENTS list | cut -c1 | sort | uniq -c | sort -rn`.
4. If directory mode, also histogram by sheet: `kicli sch $ARGUMENTS list | awk -F'\t' '{print $6}' | sort | uniq -c`.

## Pass 2 — Connectivity

5. `kicli sch $ARGUMENTS view` — full pin+net map. On a directory it walks every sheet and resolves nets via the root schematic's `kicad-cli` netlist, so sheet-pin ↔ hierarchical-label bridges (e.g. a child's `P3V3` to the root's `+3.3V`) appear under a single net name.
6. Floating pins: `kicli sch $ARGUMENTS view | grep '→ ~'` — each one is an ERC candidate. Decide: intentional no-connect, missed wire, or bug.
7. Per-IC power sanity: for each `Uxx`, run `kicli sch $ARGUMENTS view | grep '/Uxx:\|^Uxx:' | grep 'pwrin'` (the `/` prefix catches the `sheet/Uxx` form emitted in directory mode).
8. Unique nets per IC: `kicli sch $ARGUMENTS view | grep '/Uxx:\|^Uxx:' | awk '{print $4}' | sort -u`.
9. Full pin table for any IC: `kicli sch $ARGUMENTS info Uxx --pins` — NAME, TYPE, and NET columns are filled (resolved across sheets when $ARGUMENTS is a dir).
10. All pins on a specific net (flat table with pin types): `kicli sch $ARGUMENTS view --net <NAME>`.

## Pass 3 — Datasheet cross-check

For each significant IC (Uxx):

9. Find its LCSC/JLCPCB code from the `PartNo` column (or `kicli sch $ARGUMENTS info Uxx`).
10. `kicli jlcpcb part <LCSC_ID>` — extract the datasheet URL.
11. Fetch the datasheet (WebFetch) and check:
    - Every power pin (VDD/VCC/VREF/AVDD/…) has a bypass cap within a reasonable distance
    - Pull-ups/downs match the recommended values
    - Crystal load caps match the resonator's spec
    - Unused inputs are tied off per datasheet recommendation
    - RESET / BOOT / strap pins are in the required state
12. Compare `kicli sch` connectivity against datasheet block diagram.

## Pass 4 — BOM cross-check

13. `kicli jlcpcb check $ARGUMENTS` — side-by-side table of schematic values/footprints vs JLCPCB API truth per component. Scan for:
    - `Match=NO` → substring heuristic caught a footprint discrepancy; open the row and verify (common false positives: `Crystal_SMD_3225-4Pin` vs `SMD3225-4P`)
    - `SchValue` ≠ implied-from-`JLCPCBModel` → wrong LCSC assigned (e.g. SchValue says `4k7` but Model is `0402WGF1002TCE` = 10k)
    - `Stock=0` → assembly won't run; find substitute via `jlcpcb search`
    - `PartNo=(unset)` → BOM incomplete; run `jlcpcb search` to fill
    Don't blindly trust Match=yes either — read both strings yourself.

## Pass 5 — ERC

14. Stream KiCad's ERC report directly: `kicli sch <root.kicad_sch> erc -o -`. For programmatic triage, use `kicli sch <root> erc -o - --format json | jq '.sheets[].violations[] | select(.severity=="error")'` to get structured JSON (schema: https://schemas.kicad.org/erc.v1.json). Both require a single `.kicad_sch`, not a directory.

## Report format

Produce a concise report with three sections:

- **Blockers** — things that will break the board (floating critical pins, wrong part package, missing decoupling on critical rail).
- **Warnings** — things that may work but are risky (tolerance mismatch, marginal bypass value, unused-input strategy).
- **OK** — one-line summary of what passed.

Cite each finding with the exact `kicli sch … view` line or datasheet section. No hand-waving.
