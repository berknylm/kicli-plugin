---
description: End-to-end KiCad schematic review — inventory, connectivity, floating pins, datasheet cross-check. Usage — /kicli:review <sch-file-or-project-dir>.
argument-hint: <file.kicad_sch | project/>
allowed-tools: Bash(kicli:*), Bash(grep:*), Bash(awk:*), Bash(sort:*), Bash(cut:*), Read, WebFetch
---

# kicli schematic review

Target: `$ARGUMENTS`

Perform a thorough, grounded review of the schematic. **Do not guess — run the commands and report what kicli shows.**

## Pass 1 — Inventory & completeness

1. `kicli sch $ARGUMENTS list` — print the component table (REF, VALUE, LIB, FOOTPRINT, PartNo).
2. Flag components with empty `PartNo` (5th column) — those block BOM generation.
3. Histogram by reference prefix: `kicli sch $ARGUMENTS list | cut -c1 | sort | uniq -c | sort -rn`.
4. Note any unusual ratios (e.g. many U's without decoupling C's nearby).

## Pass 2 — Connectivity

5. `kicli sch $ARGUMENTS view` — full pin+net map.
6. Floating pins: `kicli sch $ARGUMENTS view | grep '→ ~'` — each one is an ERC candidate. Decide: intentional no-connect, missed wire, or bug.
7. Per-IC power sanity: for each `Uxx`, run `kicli sch $ARGUMENTS view | grep '^Uxx:' | grep 'pwrin'` and list the power nets that reach it.
8. Unique nets per IC: `kicli sch $ARGUMENTS view | grep '^Uxx:' | awk '{print $4}' | sort -u`.

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

## Pass 4 — ERC

13. `kicli sch $ARGUMENTS erc` — KiCad's own electrical rule check. Triage output.

## Report format

Produce a concise report with three sections:

- **Blockers** — things that will break the board (floating critical pins, wrong part package, missing decoupling on critical rail).
- **Warnings** — things that may work but are risky (tolerance mismatch, marginal bypass value, unused-input strategy).
- **OK** — one-line summary of what passed.

Cite each finding with the exact `kicli sch … view` line or datasheet section. No hand-waving.
