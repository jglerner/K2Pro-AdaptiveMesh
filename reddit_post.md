# Creality K2 Pro – Adaptive Bed Mesh macro (no KAMP needed)

Following my previous post about [SCREWS_TILT_ADJUST](https://www.reddit.com/r/Creality_k2/comments/1ryve1k/a_creality_k2_pro_screws_tilt_adjust_macro_based/), here is another macro I've been working on: **ADAPTIVE_BED_MESH**.

---

## The problem

The K2 Pro PRTouch probe uses a **spiral pattern starting from the G28 point outward**, which means only odd NxN grids are valid: **5x5, 7x7, or 9x9**. Most of us end up with a fixed 9x9 mesh (81 probe points) for every single print — even a small 50x50mm object.

I tried installing KAMP but it always fell back to 9x9 regardless. After some investigation I found that KAMP's complexity wasn't necessary — Orca Slicer already knows the print area and can pass it directly to a macro.

---

## The solution

A simple macro that:
- Receives the print area coordinates from Orca Slicer
- Automatically picks **5x5, 7x7 or 9x9** based on the largest side of the print area
- Adds a 10mm padding around the area and clamps to bed limits
- Falls back to full 9x9 safely if called without parameters

| Print area (longest side) | Mesh | Probe points |
|---------------------------|------|--------------|
| ≤ 150mm | 5x5 | 25 |
| ≤ 220mm | 7x7 | 49 |
| > 220mm | 9x9 | 81 |

No KAMP, no Moonraker plugins, no `[exclude_object]` dependency.

---

## The macro

Add this to your `macro.cfg`:

```jinja2
[gcode_macro ADAPTIVE_BED_MESH]
description: Auto-selects 5x5, 7x7 or 9x9 mesh based on print area
gcode:
    {% set bed_min = 10 %}
    {% set bed_max = 290 %}
    {% set padding = 10 %}

    {% set min_x = [params.PRINT_MIN_X|default(bed_min)|float - padding, bed_min]|max %}
    {% set min_y = [params.PRINT_MIN_Y|default(bed_min)|float - padding, bed_min]|max %}
    {% set max_x = [params.PRINT_MAX_X|default(bed_max)|float + padding, bed_max]|min %}
    {% set max_y = [params.PRINT_MAX_Y|default(bed_max)|float + padding, bed_max]|min %}

    {% set span = [max_x - min_x, max_y - min_y]|max %}

    {% if span <= 150 %}
        {% set count = 5 %}
    {% elif span <= 220 %}
        {% set count = 7 %}
    {% else %}
        {% set count = 9 %}
    {% endif %}

    {action_respond_info("Adaptive mesh: %dx%d | area (%.0f,%.0f)-(%.0f,%.0f)" % (count, count, min_x, min_y, max_x, max_y))}
    BED_MESH_CALIBRATE PROFILE=default MESH_MIN={min_x},{min_y} MESH_MAX={max_x},{max_y} PROBE_COUNT={count},{count}
```

---

## Changes to START_PRINT

In your `START_PRINT` macro, replace:
```
BED_MESH_PROFILE LOAD=default
```
With:
```jinja2
ADAPTIVE_BED_MESH PRINT_MIN_X={params.PRINT_MIN_X|default(10)|float} PRINT_MIN_Y={params.PRINT_MIN_Y|default(10)|float} PRINT_MAX_X={params.PRINT_MAX_X|default(290)|float} PRINT_MAX_Y={params.PRINT_MAX_Y|default(290)|float}
```

---

## Changes to Orca Slicer

In **Printer Settings → Machine G-code → Machine start G-code**, update your `START_PRINT` line to pass the print area:

```
START_PRINT EXTRUDER_TEMP=[initial_layer_temperature] BED_TEMP=[bed_temperature] PRINT_MIN_X={first_layer_print_min[0]} PRINT_MIN_Y={first_layer_print_min[1]} PRINT_MAX_X={first_layer_print_max[0]} PRINT_MAX_Y={first_layer_print_max[1]}
```

Orca Slicer already calculates `first_layer_print_min` and `first_layer_print_max` — we're just making use of what was already there.

---

## Verification

At the start of each print, check the Klipper console. You should see a line like:
```
Adaptive mesh: 5x5 | area (148,150)-(248,250)
```

---

## Notes

- Tested on Creality K2 Pro with Cryo Tack bed (PLA at 35°C bed / 220°C nozzle)
- The K2 Pro bed has a natural dome shape (~0.65mm center-to-edge with standard aluminium plate) — the bicubic mesh handles this well regardless of which grid size is selected
- This macro does a **live probe at every print start** instead of loading a saved profile — the benefit is a fresh, area-accurate mesh every time

---

Hope this helps! Happy to answer questions or improve it based on feedback.
