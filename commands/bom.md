---
description: Generate a JLCPCB-ready BOM from a KiCad schematic or project directory, fill any missing part numbers, and export the final CSV. Usage — /kicli:bom <file-or-dir> [output.csv].
argument-hint: <file.kicad_sch | project/> [output.csv]
allowed-tools: Bash(kicli*), Bash(grep*), Bash(awk*), Read
---

# kicli BOM preparation

Target: `$0`
Output CSV (optional): `$1`

## Steps

### 1. Initial scan

```
kicli jlcpcb bom "$0"
```

This merges every `.kicad_sch` in the directory (or handles a single file) and emits a CSV to stdout. Rows with empty `LCSC` column need attention. A summary is printed to stderr — capture both:

```
kicli jlcpcb bom "$0" 2>&1 >/dev/null
```

### 2. Fill missing parts

For each component value that lacks a PartNo:

1. `kicli jlcpcb search "<value> <package>" --basic --in-stock -n 5`
2. Pick a part **with circuit context in mind** — don't just take the first hit. Consider:
   - Voltage rating (cap dielectric, resistor power)
   - Temperature coefficient (X7R vs Y5V for caps)
   - Tolerance (critical for timing resistors, dividers)
   - Package fit with existing footprint
3. `kicli jlcpcb part <LCSC_ID>` — verify brand, stock, datasheet before committing.
4. Bulk-assign by value + footprint filter:
   ```
   kicli sch "$0" set-all "<value>" LCSC <LCSC_ID> --footprint "*<pkg>*" --dry-run
   ```
   Review the dry-run output, then re-run without `--dry-run`.

### 3. Verify completeness

```
kicli jlcpcb bom "$0" | grep ',$' && echo "INCOMPLETE" || echo "READY"
```

### 4. Export

If `$1` was provided:
```
kicli jlcpcb bom "$0" -o "$1"
```

Otherwise ask the user for an output path, then run the same command.

## Guardrails

- **Never assign a part without checking the datasheet** — circuit context matters (e.g. same 100nF cap, but at 3.3V rail vs 48V rail needs different voltage rating).
- **Always use `--footprint` glob for bulk edits** — otherwise a 100nF 0402 change may rewrite the 100nF 1206 bulk caps too.
- **Always `--dry-run` first on `set-all`.** Show the user the expected changes, then commit.
- Prefer JLCPCB `basic` parts to avoid the $3 extended-part assembly fee per unique part (unless the user explicitly wants extended).
