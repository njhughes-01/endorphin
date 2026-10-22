---
layout: default
title: Rear Belt Clip Kinematics
nav_order: 5
permalink: docs/rear-belt-kinematics
---

# Rear Belt Clips — Diagonal Motion Fix

If your belt clips are attached to the **rear posts** of the printhead (EVA or Vz), the printer will move diagonally when commanded to move in Y only. This page explains why, and how to fix it with a drop-in Klipper kinematics module.

- TOC
{:toc}

---

## The symptom

- Jogging Y in Mainsail/Fluidd moves the toolhead diagonally (~45°) instead of straight back
- X-only moves look correct
- Homing may fail or crash into a corner

## Why it happens

The stock `hybrid_corexy` kinematics assumes the C-belt is attached at the **front** of the carriage. The Markforged/hybrid_corexy math works like this:

- During a Y-only move, the **X motor must compensate** to hold the toolhead's X position
- The direction of that compensation depends on which side of the carriage the belt pulls from

When belt clips are at the **back**, the belt's mechanical action is reversed:

| Clip position | Pulling belt toward X motor causes carriage to... |
|:---|:---|
| Front (stock assumption) | Move **away** from X motor ✓ |
| Rear | Move **toward** X motor ✗ |

This inversion means the X compensation runs in the wrong direction, producing diagonal motion. The Marlin equivalent of this inversion has a dedicated flag: `#define MARKFORGED_INVERSE`.

## The fix (Klipper)

A custom kinematics module inverts the X↔Y coupling sign. Two values change together — they are forward and inverse transforms of each other and must stay in sync:

| Location | Stock `hybrid_corexy` | Rear-belt fix |
|:---|:---|:---|
| C itersolve flag | `b'-'` | `b'+'` |
| `calc_position` | `pos[0] + pos[1]` | `pos[0] - pos[1]` |

{: .note }
The module only runs on the Klipper host (the Pi). The firmware on the printer's board stays stock Klipper — there is nothing to recompile or reflash for this fix.

### Installation

1. Copy [`hybrid_corexy_rear_belt.py`](https://github.com/njhughes-01/endorphin/blob/main/klipper/kinematics/hybrid_corexy_rear_belt.py) into your Klipper kinematics directory:

   ```bash
   # On your Klipper Pi:
   cp hybrid_corexy_rear_belt.py ~/klipper/klippy/kinematics/
   # keep Klipper's updater from flagging the repo as dirty
   echo klippy/kinematics/hybrid_corexy_rear_belt.py >> ~/klipper/.git/info/exclude
   ```

2. In `printer.cfg`, change the kinematics line:

   ```yml
   [printer]
   kinematics: hybrid_corexy_rear_belt   # was: hybrid_corexy
   max_velocity: 250
   max_accel: 4000
   max_z_velocity: 10
   max_z_accel: 100
   ```

3. Set your X/Y travel. The Endorphin homes to **X max and Y max** (the rear-right corner), so `position_endstop` equals `position_max` on both axes. The Y limit depends on the printhead:

   | Printhead | `stepper_y` `position_endstop` / `position_max` |
   |:---|:---|
   | EVA | 190 |
   | Vz | 200 (stock guide value — measure yours, see below) |

   ```yml
   [stepper_x]
   position_endstop: 220
   position_max: 220

   [stepper_y]
   position_endstop: 200
   position_max: 200
   ```

4. Save and restart Klipper (Firmware Restart in Mainsail).

{: .warning }
The Klipper module is not updated by Moonraker's update manager. If a Klipper update ever fails to load `hybrid_corexy_rear_belt`, copy the file back into `~/klipper/klippy/kinematics/` and restart.

### Verification

After switching, **test both motion and position reporting** — a config with only one sign flipped will pass a jog test but corrupt homing:

1. **Jog +Y 20mm** — toolhead should move straight back, zero X drift
2. **Jog +X 20mm** — toolhead should move right, zero Y drift
3. **Home all axes** — should home cleanly to the rear-right without crashing
4. **Run `GET_POSITION`** in the Klipper console — reported X/Y should match the physical position

If Y still moves diagonally after this change, your belt clips are at the front and you should revert to stock `hybrid_corexy`.

If X homes *away* from its switch, flip that motor's direction (`dir_pin: PB12` ↔ `dir_pin: !PB12`). Fix motor directions with the kinematics already switched — the two interact.

---

## Measuring travel and the origin

Homing to the rear-right does **not** move the origin. `0,0` is still the **front-left** corner, which is what every slicer assumes. Leave your slicer's origin at 0,0 front-left; if you move it to the rear-right, the slicer outputs negative coordinates and Klipper rejects the print with "Move out of range".

Because both axes home to max, the number you put in `position_endstop` decides where the front-left end of travel is. To measure:

1. Home X and Y.
2. Jog toward the front-left in small steps, watching the printhead.
3. Stop just before anything touches. Travel is `position_endstop` minus where you stopped. Adjust and re-test until 0,0 lands where you want it.
4. Copy the final X/Y travel into your slicer's printable area so the two agree.

Klipper will never move past `position_max`, and on a max-homing axis `position_max` must equal the switch position. If the printhead (or the probe on it) gets close to the frame at the rear-right, the only fix is to move the switch, not a config change.

---

## Probe setup after the conversion

If you run a BLTouch on the new printhead, re-derive every probe setting from scratch — the stock Ender 5 values are all wrong now.

- **Measure the X/Y offset; don't trust drawings.** Put a dot on the bed, jog the nozzle over it and write down X/Y, then jog until the probe pin is over the same dot and write that down. `x_offset = nozzle X − probe X` and `y_offset = nozzle Y − probe Y`. (On the build below, the published mount drawing was off by more than 50 mm; the measured offset was `-28, -11`.)
- **`home_xy_position`** in `[safe_z_home]` is the *nozzle* position that puts the *probe* at the bed centre: `bed centre − probe offset`.
- **`mesh_min` / `mesh_max`** are *probe* coordinates. Keep them where the pin can reach and still lands on the bed. If the pin ever misses the bed, it never triggers, and the nozzle reaches the bed first.
- **`horizontal_move_z` must be greater than the probe `z_offset`.** A BLTouch that rides high (large `z_offset`) makes Klipper refuse to mesh with "horizontal_move_z can't be less than probe's z_offset".
- **`PROBE_CALIBRATE` may need a negative `position_min` on Z** to get the nozzle down to the paper. If you see "Move out of range" while lowering with `TESTZ`, lower `stepper_z` `position_min` and try again.

---

## Tested build

Running config: [`ender5-endorphin-vz-sherpa-skr-mini-e3-v2.cfg`](https://github.com/njhughes-01/endorphin/blob/main/klipper/config/ender5-endorphin-vz-sherpa-skr-mini-e3-v2.cfg)

| Part | Setting |
|:---|:---|
| Printhead | Vz Printhead, **rear** belt clips, `kinematics: hybrid_corexy_rear_belt` |
| Board | BTT SKR Mini E3 V2.0, stock Klipper firmware (STM32F103, 28KiB bootloader, USB) |
| Travel | X 235 / Y 207 (homes to X max, Y max; measured) |
| Motor directions | X `PB12`, Y `!PB2`, Z `!PC5` |
| Extruder | Sherpa Mini, pancake motor: `rotation_distance: 22.6789511`, `gear_ratio: 50:10`, `run_current: 0.6`, `dir_pin: PB4` |
| Hotend | V6 clone with the stock Creality thermistor (`EPCOS 100K B57560G104F`) |
| Probe | BLTouch, `x_offset: -28`, `y_offset: -11`, `horizontal_move_z: 9`, `stow_on_each_sample: False` |
| Print start | Adaptive mesh (`BED_MESH_CALIBRATE ADAPTIVE=1`) plus an adaptive purge line in front of the part |

The config also has `START_PRINT`/`END_PRINT` aliases, so Orca/Elegoo Slicer (`START_PRINT BED_TEMP=… EXTRUDER_TEMP=…`) and Cura (`PRINT_START BED=… EXTRUDER=…`) both work.

---

## The fix (Marlin)

Marlin has a built-in flag for this. In `Configuration.h`:

```c
#if ANY(MARKFORGED_XY, MARKFORGED_YX)
  #define MARKFORGED_INVERSE  // Enable for an inverted Markforged kinematics belt path
#endif
```

See the [Firmware page](/docs/firmware) for full Marlin flashing instructions.

---

## Why builders end up with rear clips

Both supported printheads have belt posts on the front **and** rear of the carriage.

- **EVA:** The default EVA CoreXY guide instructs users to print `core_xy_belt_grabber.stl`, which attaches the belt to the front of the carriage. The Endorphin guide replaces this with `eva-endorphin_belt_grabber.stl` and notes that `face_belt_grabber.stl` is not needed — but it does not specify whether the clip goes on the front or rear posts. Builders who follow the EVA belt-routing video often end up with rear clips.
- **Vz:** The [stage 3 guide](/docs/stages/stage-3) lets you choose front or rear posts (just not one of each).

Either placement is mechanically valid — rear clips just require the inverted kinematics.

**Front clips** → use stock `kinematics: hybrid_corexy`  
**Rear clips** → use `kinematics: hybrid_corexy_rear_belt`

If you are starting fresh and have not yet attached the belt, front clips are the simpler option (no custom kinematics needed).
