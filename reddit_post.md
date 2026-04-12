# Creality K2 Pro – Adaptive Bed Mesh macro (no KAMP needed)

Following my previous post about [SCREWS_TILT_ADJUST](https://www.reddit.com/r/Creality_k2/comments/1ryve1k/a_creality_k2_pro_screws_tilt_adjust_macro_based/), here is another macro I've been working on: **ADAPTIVE_BED_MESH**.

Full files and future updates: **https://github.com/jglerner/K2Pro-AdaptiveMesh**

---

## The problem

The K2 Pro PRTouch probe uses a **spiral pattern starting from the G28 point outward**. Most of us end up with a fixed 9x9 mesh (81 probe points) over the full 300x300mm bed for every single print — even a small 50x50mm object. That means ~35mm spacing between probe points, which is coarse.

I tried KAMP but it always fell back to 9x9 regardless. After some investigation I found that KAMP's complexity wasn't necessary — Orca Slicer already knows the print area and can pass it directly to a macro.

---

## Important discovery about PRTouch

During testing I found that the K2 Pro's PRTouch firmware (`prtouch_v3_wrapper.py`) **hardcodes 81 probe points (9x9)**. Passing a different PROBE_COUNT at runtime causes:

```
IndexError: list index out of range
```

So the probe count cannot be changed dynamically. However, **the mesh boundaries can still adapt to the print area**, and this makes a significant difference:

| Full bed 9x9 | Adaptive area 9x9 |
|---|---|
| ~35mm spacing over 300x300mm | ~12mm spacing over 100x100mm |
| Same probe time | Same probe time |
| Coarse mesh everywhere | Dense accurate mesh where you print |

Real result: **Range dropped from 0.65mm (full bed) to 0.153mm (adaptive area)** for the same print.

---

## The macro

Add this to your `macro.cfg`:

    [gcode_macro ADAPTIVE_BED_MESH]
    description: Adapts mesh area to print size, always uses 9x9 (PRTouch requirement)
    gcode:
        {% set bed_min = 10 %}
        {% set bed_max = 290 %}
        {% set padding = 10 %}

        {% set min_x = [params.PRINT_MIN_X|default(bed_min)|float - padding, bed_min]|max %}
        {% set min_y = [params.PRINT_MIN_Y|default(bed_min)|float - padding, bed_min]|max %}
        {% set max_x = [params.PRINT_MAX_X|default(bed_max)|float + padding, bed_max]|min %}
        {% set max_y = [params.PRINT_MAX_Y|default(bed_max)|float + padding, bed_max]|min %}

        {action_respond_info("Adaptive mesh: 9x9 | area (%.0f,%.0f)-(%.0f,%.0f)" % (min_x, min_y, max_x, max_y))}
        BED_MESH_CALIBRATE PROFILE=default MESH_MIN={min_x},{min_y} MESH_MAX={max_x},{max_y} PROBE_COUNT=9,9

---

## Changes to START_PRINT

In your `START_PRINT` macro, replace:

    BED_MESH_PROFILE LOAD=default

With:

    ADAPTIVE_BED_MESH PRINT_MIN_X={params.PRINT_MIN_X|default(10)|float} PRINT_MIN_Y={params.PRINT_MIN_Y|default(10)|float} PRINT_MAX_X={params.PRINT_MAX_X|default(290)|float} PRINT_MAX_Y={params.PRINT_MAX_Y|default(290)|float}

**Important:** this must be on a single line with no line break.

---

## Changes to Orca Slicer

In **Printer Settings → Machine G-code → Machine start G-code**, update your `START_PRINT` line:

    START_PRINT EXTRUDER_TEMP=[first_layer_temperature] BED_TEMP=[bed_temperature]

---

## Verification

At the start of each print, check the Klipper console:
```
Adaptive mesh: 9x9 | area (122,109)-(178,290)
```

---

## Gotcha: gcode_macro.cfg first line

The K2 Pro's `gcode_macro.cfg` starts with `START# F012`. Klipper's config parser sees `START` as an invalid key and throws:
```
File contains no section headers, line 1: 'START\n'
```
Fix it via SSH:
```bash
sed -i '1s/^START/# START/' /mnt/UDISK/printer_data/config/gcode_macro.cfg
```

---

## Notes

- Tested on Creality K2 Pro with Cryo Tack bed (PLA at 35°C bed / 220°C nozzle)
- No KAMP, no Moonraker plugins, no `[exclude_object]` dependency
- Falls back to full bed 9x9 if called without parameters

Hope this helps! Happy to answer questions or improve it based on feedback.
