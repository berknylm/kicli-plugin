---
description: Search JLCPCB for a part and present the best candidates with stock, price, and datasheet. Usage — /kicli:search <query> [-n N] [--basic|--extended] [--in-stock] [--package PKG].
argument-hint: <query> [-n N] [--basic|--extended] [--in-stock] [--package PKG]
allowed-tools: Bash(kicli jlcpcb:*), Bash(grep:*), Bash(awk:*)
---

# kicli JLCPCB part search

Query: `$ARGUMENTS`

## Steps

1. Run:
   ```
   kicli jlcpcb search $ARGUMENTS
   ```
   If the user did not include `-n`, default to `-n 10` for brevity.
2. If the user did not narrow by tier or stock and the query is for a commodity passive (resistor / capacitor), prefer:
   ```
   kicli jlcpcb search "<query>" --basic --in-stock -n 10
   ```
   Basic parts avoid the per-unique-part assembly fee on JLCPCB's PCBA service.
3. For the top 3 candidates, run `kicli jlcpcb part <LCSC_ID>` to pull detail (brand, stock, datasheet URL).
4. Present a short comparison: `LCSC | Package | Tolerance/Rating | Stock | Price | Tier`. Call out any candidate that is obsolete, EOL, or low-stock (<1000).
5. **Do not pick for the user.** The right part depends on voltage, temperature, tolerance, and physical constraints that only the schematic + datasheet reveal. List the candidates and let the user decide — or ask clarifying questions about the circuit context.

## Common query patterns

```
100nF 0402              # capacitor value + package
10k 0603 1%             # resistor with tolerance
LM358                   # op-amp part number
STM32G0 LQFP            # MCU family with package
RP2040                  # exact chip name
```

## Flags quick-ref

- `--basic` — JLCPCB basic parts only (no extended-part fee)
- `--extended` — extended-only
- `--in-stock` — exclude zero-stock rows
- `--package <pkg>` — restrict by package (e.g. `0402`, `SOIC-8`)
- `-n N` — number of results (default unlimited; prefer 10)
