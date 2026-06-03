---
layout: default
title: Rear Belt Clip Kinematics
nav_order: 5
permalink: docs/rear-belt-kinematics
---

# Rear Belt Clips — Diagonal Motion Fix

If you followed the default EVA assembly instructions and attached your belt clips to the **rear posts** of the printhead, you will see the printer move diagonally when commanded to move in Y only. This page explains why, and how to fix it with a drop-in Klipper kinematics module.

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
| Rear (EVA default) | Move **toward** X motor ✗ |

This inversion means the X compensation runs in the wrong direction, producing diagonal motion. The Marlin equivalent of this inversion has a dedicated flag: `#define MARKFORGED_INVERSE`.

## The fix (Klipper)

A custom kinematics module inverts the X↔Y coupling sign. Two values change together — they are forward and inverse transforms of each other and must stay in sync:

| Location | Stock `hybrid_corexy` | Rear-belt fix |
|:---|:---|:---|
| C itersolve flag | `b'-'` | `b'+'` |
| `calc_position` | `pos[0] + pos[1]` | `pos[0] - pos[1]` |

### Installation

1. Copy [`hybrid_corexy_rear_belt.py`](https://github.com/njhughes-01/endorphin/blob/main/klipper/kinematics/hybrid_corexy_rear_belt.py) into your Klipper kinematics directory:

   ```bash
   # On your Klipper Pi:
   cp hybrid_corexy_rear_belt.py ~/klipper/klippy/kinematics/
   ```

2. In `printer.cfg`, change the kinematics line:

   ```yaml
   [printer]
   kinematics: hybrid_corexy_rear_belt   # was: hybrid_corexy
   max_velocity: 250
   max_accel: 4000
   max_z_velocity: 5
   max_z_accel: 100
   ```

3. If using the EVA printhead, ensure Y is limited to 190:

   ```yaml
   [stepper_y]
   position_endstop: 190
   position_max: 190
   ```

4. Save and restart Klipper (Firmware Restart in Mainsail).

### Verification

After switching, **test both motion and position reporting** — a config with only one sign flipped will pass a jog test but corrupt homing:

1. **Jog +Y 20mm** — toolhead should move straight back, zero X drift
2. **Home all axes** — should home cleanly without crashing
3. **Run `GET_POSITION`** in the Klipper console — reported X/Y should match the physical position

If Y still moves diagonally after this change, your belt clips are at the front and you should revert to stock `hybrid_corexy`.

---

## The fix (Marlin)

Marlin has a built-in flag for this. In `Configuration.h`:

```c
#if ANY(MARKFORGED_XY, MARKFORGED_YX)
  #define MARKFORGED_INVERSE  // Enable for an inverted Markforged kinematics belt path
#endif
```

See the [Firmware page](docs/firmware) for full Marlin flashing instructions.

---

## Why the EVA instructions lead here

The default EVA CoreXY guide instructs users to print `core_xy_belt_grabber.stl`, which attaches the belt to the front of the carriage. The Endorphin guide replaces this with `eva-endorphin_belt_grabber.stl` and notes that `face_belt_grabber.stl` is not needed — but it does not specify whether the clip goes on the front or rear posts.

Since the EVA carrier has posts on both front and rear, builders who follow the EVA belt-routing video will often end up with rear clips. This is mechanically valid — it just requires the inverted kinematics.

**Front clips** → use stock `kinematics: hybrid_corexy`  
**Rear clips** → use `kinematics: hybrid_corexy_rear_belt`

If you are starting fresh and have not yet attached the belt, front clips are the simpler option (no custom kinematics needed).
