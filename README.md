# K2Pro-AdaptiveMesh

A Klipper macro for the **Creality K2 Pro** that runs a **9×9 bed mesh calibration sized to the model's footprint**, with a 9 mm margin on each side.

No KAMP plugin. No Moonraker dependency. Uses Klipper's built-in `[exclude_object]` module (already present on the K2 Pro).

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

The macro derives the print area from Klipper's `[exclude_object]` data — the same object polygon information OrcaSlicer emits when "Label objects" is enabled. This avoids passing slicer vector variables (`{first_layer_print_min[0]}`) in machine start G-code, which OrcaSlicer rejects with a parse error before the file ever reaches the printer.

**Area detection priority:**

1. `printer.exclude_object.objects` polygon data (preferred)
2. Explicit `PRINT_MIN_X/Y` + `PRINT_MAX_X/Y` params (legacy / manual calls)
3. Full 10–290 mm bed mesh (no area data available)

**Then the macro:**

1. **Preheats** — bed to **40 °C**, extruder to **140 °C** — before probing.
   - 40 °C stabilises the bed surface geometry without the full thermal expansion of print temperatures.
   - 140 °C softens any residual filament on the nozzle so it does not drag across the surface, but is too cold to drip.
2. Expands the model bounding box by **9 mm on each side**, clamped to bed limits (10–290 mm).
3. Runs `BED_MESH_CALIBRATE` over that region with `PROBE_COUNT=9,9`.
4. Saves the result to the `default` profile.

After the macro completes, `START_PRINT` heats to full print temperatures and begins the print.

---

## Installation

### 1. Verify `[exclude_object]` is in printer.cfg

The K2 Pro ships with this already. Confirm with Ctrl+F in your config editor:

```ini
[exclude_object]
```

If it is missing, add it as a standalone section anywhere in `printer.cfg`.

### 2. Add the macro

Copy `adaptive_bed_mesh.cfg` to `/mnt/UDISK/printer_data/config/` and add this line to `printer.cfg`:

```ini
[include adaptive_bed_mesh.cfg]
```

Or copy the contents directly into your `macro.cfg`.

### 3. Update START_PRINT

In your `START_PRINT` macro, replace:

```
BED_MESH_PROFILE LOAD=default
```

With:

```
ADAPTIVE_BED_MESH
```

No parameters needed — the macro reads the print area automatically.

> Remove any `G28` that previously ran before `BED_MESH_PROFILE LOAD=default` in `START_PRINT`. The macro handles homing internally.

### 4. Enable "Label objects" in OrcaSlicer

In **Print Settings → Others**, turn on **Label objects** (also called "Identify objects").

This makes OrcaSlicer emit `EXCLUDE_OBJECT_DEFINE` statements at the top of each G-code file. Klipper processes these before executing machine start G-code, which is how the macro knows the print area.

### 5. Update OrcaSlicer machine start G-code

In **Printer Settings → Machine G-code → Machine start G-code**, use:

```
START_PRINT EXTRUDER_TEMP=[initial_layer_temperature] BED_TEMP=[bed_temperature]
```

**Do not** pass `{first_layer_print_min[0]}` or similar vector variables here — OrcaSlicer's machine start G-code parser rejects that syntax and errors before generating the file.

### 6. Restart Klipper

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
- Requires OrcaSlicer with **Label objects** enabled
- Requires `[exclude_object]` in `printer.cfg` (present by default on the K2 Pro)
- Falls back to full bed 9×9 if no object data is available

## License

MIT
