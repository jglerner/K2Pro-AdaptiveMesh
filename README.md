# K2Pro Adaptive Bed Mesh

## What it does
Automatically selects the correct mesh density based on the actual print area, instead of always running a full 9x9 mesh.

| Print area (longest side) | Mesh used | Probe points |
|---------------------------|-----------|--------------|
| ≤ 150mm                   | 5x5       | 25           |
| ≤ 220mm                   | 7x7       | 49           |
| > 220mm                   | 9x9       | 81           |

A 10mm padding is added around the print area, clamped to bed limits (10-290mm).

## Why this instead of KAMP
- No `[exclude_object]` dependency
- No Moonraker plugin needed
- Falls back safely to full 9x9 if called without parameters
- Orca Slicer already provides `{first_layer_print_min}` / `{first_layer_print_max}` — no extra tooling required

## Context
- Printer: Creality K2 Pro
- Bed: Cryo Tack (PLA at 35°C bed / 220°C nozzle)
- PRTouch probe uses spiral pattern from center outward — only odd NxN grids are valid (5x5, 7x7, 9x9)
- Bed has a natural dome shape (~0.65mm center-to-edge), well compensated by bicubic mesh

## Installation
1. Copy the macro from `adaptive_bed_mesh.cfg` into your `macro.cfg`
2. In `START_PRINT`, replace `BED_MESH_PROFILE LOAD=default` — see `START_PRINT_change.txt`
3. In Orca Slicer → Printer Settings → Machine G-code → Machine start G-code, update the `START_PRINT` line — see `orca_start_gcode.txt`
4. Restart Klipper

## Verification
Check the Klipper console at print start — you should see a line like:
```
Adaptive mesh: 5x5 | area (148,150)-(248,250)
```
