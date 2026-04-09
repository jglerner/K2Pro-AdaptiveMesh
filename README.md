# K2Pro-AdaptiveMesh

A Klipper macro for the **Creality K2 Pro** that runs a **9×9 bed mesh calibration sized to the model's footprint**, with a 9 mm margin on each side.

No KAMP. No Moonraker plugin. No `[exclude_object]` dependency.

![Adaptive bed mesh result](Bed_meshCalibrate.png)

---

## The problem

The K2 Pro PRTouch probe generates a fixed 81-point (9×9) spiral pattern from center outward. Most users end up probing the full 300×300 mm bed for every print — even a 50×50 mm object — giving ~35 mm spacing between probe points. That is coarse.

KAMP does not help here: the K2 Pro's PRTouch firmware (`prtouch_v3_wrapper.py`) **hardcodes 81 probe points**. Passing a different `PROBE_COUNT` at runtime causes:

```
IndexError: list index out of range
```

The probe count cannot change. But **the mesh boundaries can** — and that makes a significant difference:

| | Full bed 9×9 | Adaptive area 9×9 |
|---|---|---|
| Spacing | ~35 mm over 300×300 mm | ~12 mm over 100×100 mm |
| Probe time | same | same |
| Quality | coarse everywhere | dense where you print |

Measured result: **range dropped from 0.65 mm (full bed) to 0.153 mm (adaptive area)** for the same print.

---

## How it works

The macro:

1. **Preheats** — bed to **40 °C**, extruder to **140 °C** — before probing.
   - 40 °C stabilises the bed surface geometry without the full thermal expansion of print temperatures.
   - 140 °C softens any residual filament on the nozzle so it does not drag across the surface, but is too cold to drip.
2. **Homes** all axes (`G28`).
3. Expands the model bounding box by **9 mm on each side**, clamped to bed limits (10–290 mm).
4. Runs `BED_MESH_CALIBRATE` over that region with `PROBE_COUNT=9,9`.
5. Saves the result to the `default` profile.

After the macro completes, `START_PRINT` heats to full print temperatures and begins the print.

---

## Installation

### 1. Add the macro

Copy the contents of `adaptive_bed_mesh.cfg` into your `macro.cfg`
(located at `/mnt/UDISK/printer_data/config/macro.cfg` on the K2 Pro).

### 2. Update START_PRINT

In your `START_PRINT` macro, replace:

```
BED_MESH_PROFILE LOAD=default
```

With (on a **single line**):

```
ADAPTIVE_BED_MESH PRINT_MIN_X={params.PRINT_MIN_X|default(10)|float} PRINT_MIN_Y={params.PRINT_MIN_Y|default(10)|float} PRINT_MAX_X={params.PRINT_MAX_X|default(290)|float} PRINT_MAX_Y={params.PRINT_MAX_Y|default(290)|float}
```

> Because the macro now includes its own `G28` and preheat, remove or move any `G28` that previously ran before `BED_MESH_PROFILE LOAD=default` in `START_PRINT`.

### 3. Update Orca Slicer

In **Printer Settings → Machine G-code → Machine start G-code**, update your `START_PRINT` line to pass the model bounding box:

```
START_PRINT EXTRUDER_TEMP=[initial_layer_temperature] BED_TEMP=[bed_temperature] PRINT_MIN_X={first_layer_print_min[0]} PRINT_MIN_Y={first_layer_print_min[1]} PRINT_MAX_X={first_layer_print_max[0]} PRINT_MAX_Y={first_layer_print_max[1]}
```

Orca Slicer calculates these coordinates automatically — they are just passed through to the macro.

### 4. Restart Klipper

---

## Verification

At the start of each print, the Klipper console should show:

```
Adaptive mesh: 9x9 | area (122,109)-(178,290)
```

---

## K2 Pro gotcha: gcode_macro.cfg first line

The K2 Pro's `gcode_macro.cfg` starts with `START# F012`. Klipper's config parser treats `START` as an invalid section header and throws:

```
File contains no section headers, line 1: 'START\n'
```

Fix it via SSH:

```bash
sed -i '1s/^START/# START/' /mnt/UDISK/printer_data/config/gcode_macro.cfg
```

---

## Files

| File | Purpose |
|------|---------|
| `adaptive_bed_mesh.cfg` | The macro — copy into `macro.cfg` |
| `orca_start_gcode.txt` | Updated Machine start G-code for Orca Slicer |
| `START_PRINT_change.txt` | Exact `START_PRINT` diff to apply |
| `Bed_meshCalibrate.png` | Example mesh result showing improved range |

---

## Compatibility

- Tested on Creality K2 Pro with Cryo Tack bed surface (PLA: 35 °C bed / 220 °C nozzle)
- Requires Orca Slicer (provides `first_layer_print_min` / `first_layer_print_max`)
- Falls back to the full bed 9×9 if called without parameters

## License

MIT
