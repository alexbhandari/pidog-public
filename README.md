# PiDog — Status, Plan & Next Steps (single living doc)

**This is the ONE source of truth. Read it first; keep it updated every session.**
Merged from the old `NAV_PLAN.md` + `AUTONOMY_PLAN.md` (both deleted) + running status.
Last updated: **2026-06-17**.

> **Public shareable mirror:** https://github.com/alexbhandari/pidog-public
> After editing this file, run `./sync_public_status.sh` to update the public copy.

---

## Goal
Navigate the room using the dog's OWN onboard camera (NO overhead cam): find/return home,
leave-and-return, eventually find-Alex and patrol without getting stuck. Record every run
(onboard cam + data) and push to GitHub so Alex can review from his phone.

## Architecture (the design decision everything follows from)
A tight visual control loop ("see target → correct heading → step → repeat") must run **where
the camera + motors are = on the Pi**; a WiFi hop per control step is fatal lag. So:

| Layer | Runs on | Job |
|---|---|---|
| Fast control + cheap eyes | **Pi** (`dogbrain.py`, ~8–10 Hz, no network) | closed-loop visual servoing, obstacle reflex, stall/fall reflex, drift self-correction |
| Smart eyes + planning | **Mac** (`nav_*.py`, ~2–3 Hz) | YOLO/landmark detection, choose bearings/goals, drive the Pi |
| Semantic judgment | **Claude** | "is that Alex?", "which doorway?", set/verify goals |
| Override + record + debug | **Mac** (`cockpit.py`, `rec_run.py`) | dashboard, recording, manual drive |

**Core principle: close the loop on VISION, never trust turn angles / gyro.** Re-sense landmarks
each step; don't dead-reckon.

---

## What WORKS (verified)
- **Persistent ORB-SLAM3 map**: load → merge → save reliably, no gdb (`slam/`). Fixed the
  YAML-comment load bug + the relocalized-shutdown save crash.
- **Stall/stuck detection** (`verified_move`): camera feature-flow + ultrasonic delta + IMU tilt
  → `moved`/`blocked`; auto-stops on a real stall. `unstick` recovery (scan→turn-to-open→forward).
- **Fall reflex**: IMU >50° off-upright → stop + refuse locomotion. Tip-over root cause fixed:
  `gait_settle()` waits for the non-blocking gait to finish before measuring (never `body_stop()`
  mid-stride).
- **Vision-measured turn** (`turn_vision`): rotation from camera feature-flow, not gyro. Calibrated
  cmd 90° → ~90° actual (vs gyro's ~30°). `PX_PER_DEG=5.0`.
- **find_home**: sweep (vision-turns) + head-scan/detect bookshelf + visual-servo center = face
  home. Verified: found shelf on sweep 1, centered at 180cm (~57% rest battery).
- **Recording → GitHub → phone**: `rec_run.py` → onboard `dog.mp4` + `data.jsonl` → `recordings/`.
- **Autonomy-off enforced in tooling** (`./dog quiet` / `./dog approach`) — can't be forgotten.

### Validated hardware / perception facts (from nav testing — build on these)
- **Gyro**: accurate at rest; UNDER-reports fast turns ~3×. Don't trust it for turn amount → use
  `turn_vision`.
- **YOLOv4-tiny** on Pi (`/detect`, ~450 ms, COCO-80): validated detecting the bookshelf ("book").
  Returns per-detection `cx` (−1..+1 horizontal bearing, 0=centered) + `area_frac` (size→closeness).
  Flaky at range → frame-smooth (require ≥2 of 3 frames).
- **Head-pan + detect** (`look`/`scan`) surveys landmarks with NO body turn (no drift, low power).
- **Ultrasonic** (`/sense distance_cm`): good for obstacle stop + approach distance; returns ≤0 on
  no-echo → median-filter and reject out-of-range.
- **Forward gait translates ~8 cm/step.** Turn gait translates too (drifts) — turns are NOT in-place;
  landmark re-centering fixes orientation but absolute position drifts.
- **Landmarks**: bookshelf = "book" (reliable forward anchor). Laptop/screen = "laptop"/"tvmonitor"
  (the home-base side, also a NO-GO object — anchor on it, never drive into it).

## Known issues / blockers
- **Battery voltage SAGS under turn current**: reads ~20% mid-turn, ~53% at rest. Dominates
  `battery_pct`. Below the sag threshold servos lose torque → turns under-rotate → `turn_vision`
  runs long/times out, walk loops bail on `<30%`. **Turn-heavy nav needs near-full charge.**
  (memory: `battery-sags-under-turn-load`)
- **Turn-drift**: turns translate the dog (not in-place) → position drifts across a multi-turn run.
- **leave-and-return**: physically executes (leaves, loops out, returns near home facing the
  bookshelf) but not yet a CLEAN verified round-trip — degraded by the battery sag.
- No compass; floor-level camera; ~10 s WiFi loop; WiFi drops when handled.

## Roadmap / next steps (priority order)
1. **Charge to FULL, re-run `home_return.py` (leave-and-return), recorded.** Should run clean now
   that `turn_vision` + timeouts are fixed and torque is available. find_home already proves the logic.
2. **Harden `turn_vision` for low torque**: bound max_steps vs command timeout; lower turn speed
   (less peak current → less sag).
3. **Fix `navigate_return.py`** to use `turn_vision` + 70 s `_curl` timeout (still has old 8 s).
4. **find-Alex** (markerless): Mac person-detection → feed bearings to the Pi servo loop; Claude
   confirms "that's Alex" from a frame; approach + stop.
5. **Patrol / explore & return**: chain find-landmark → approach (stall+recovery built) across a few
   landmarks; return home via `find_home` / SLAM relocalization.
6. **Hardware**: a ~$3 **QMC5883L magnetometer** gives absolute heading and removes the turn-angle
   problem entirely — highest-leverage upgrade. (Optional earlier idea: AprilTag on the charger as a
   reliable "home" beacon to bootstrap closed-loop control without perception risk.)

## Done (history)
- Bug fixes, LED battery bar, scan+pitch, battery monitoring, clearance turn.
- ORB-SLAM3 built + runs on the Pi; persistent map working.
- Detection (YOLO), head-scan survey, frame-smoothing, visual-servo turn — all validated.
- Stall detection, fall reflex, vision-measured turn, find_home, per-run recorder.

## Key files
- `dogbrain.py` (Pi server): `verified_move`, `turn_vision`, fall reflex, `/act /sense /detect /scan /imu`.
- `./dog` helper: `sense|see|perceive|scan|quiet|approach|unstick|up|health`.
- `nav_approach.py` (visual-servo approach + `unstick`), `find_home.py`, `home_return.py`,
  `navigate_return.py`, `rec_run.py` (recorder), `slam/` (SLAM), `cockpit/` (dashboard).
- Memory: `pidog-status`, `battery-sags-under-turn-load`, `disable-autonomy-before-nav`,
  `dont-claim-dog-cant-move`, `no-unbounded-waits`, `web-search-when-stuck`.

## Recordings (GitHub — watch on phone)
- find_home (success): `recordings/2026-06-16_find_home_onboard.mp4`
- leave-and-return (degraded by battery sag): `recordings/2026-06-16_leave_and_return_onboard.mp4`
