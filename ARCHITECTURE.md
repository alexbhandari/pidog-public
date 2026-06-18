# PiDog — System Architecture (what's actually running)

How the running system is wired today, on the **Dog** (Raspberry Pi) and the **Mac**. For the
goals/plan see [OVERVIEW.md](OVERVIEW.md) / [STATUS.md](STATUS.md); this doc is the *machinery*.

## The one-line picture
The **Dog is the body**: a Raspberry Pi runs one always-on program (`dogbrain.py`) that owns all
the hardware (legs, camera, sensors) and exposes it over HTTP. The **Mac is the brain**: Claude
(the AI) and a dashboard read the dog's sensors/camera, decide what to do, and send it commands.
The **fast, reflex-level control runs on the Dog itself** so it never waits on Wi-Fi; the Mac
sends *goals*, not per-step twitches.

```
            Wi-Fi (HTTP / SSH)
  MAC  <------------------------->  DOG (Raspberry Pi)
  - Claude (AI brain, high level)    - dogbrain.py  (HTTP server :8080)
  - cockpit dashboard  :8088           owns: legs, camera, ultrasonic, IMU, LED
  - ./dog helper + nav scripts       - ORB-SLAM3 pipeline  (:8081, when mapping)
  - run recorder
```

---

## On the DOG (Raspberry Pi) — `dogbrain.py`
One Python program, a multithreaded HTTP server on `0.0.0.0:8080`. It is the *only* thing that
touches the hardware (SunFounder PiDog library for servos/ultrasonic/IMU/touch/LED, and
`picamera2` for a 320×240 camera). Kept dependency-light to fit the Pi's RAM.

### HTTP endpoints (how the Mac talks to the body)
| Endpoint | What it does |
|---|---|
| `GET /health` | liveness check |
| `GET /sense` | one JSON telemetry blob: distance (ultrasonic), IMU accel/gyro, touch, sound, **battery (v + %)**, heading, odometry, `fallen`/`blocked` flags, RAM |
| `GET /frame.jpg` | latest camera frame (JPEG) |
| `GET /detect` | runs **YOLOv4-tiny** (COCO 80-class) on the current frame → objects with `cx` (left/right bearing) + `area_frac` (closeness) |
| `GET /scan` | pans the head across yaw angles, returns a **depth profile** (distance at each angle) + frames |
| `GET /imu` | high-rate IMU stream (for SLAM fusion) |
| `POST /act` | execute one action (the command bus, below) |

### Background threads (always running)
- **`reflex_loop` (20 Hz)** — safety reflexes: hard-stop forward motion if an obstacle is `<18 cm`;
  **fall watchdog** that flags `fallen` when the IMU says the body tipped `>50°` off upright (and
  auto-clears when righted).
- **`behavior_loop`** — optional "alive" autonomy (idle head glances, tail, tilt). Toggled off for
  manual/auto driving.
- **`heading_loop`** — integrates the gyro into a continuous heading estimate.
- **`imu_sampler` (~250 Hz)** — fills an IMU buffer that the SLAM pipeline reads.

### The command bus: `POST /act {"cmd": ...}`
- **Locomotion:** `forward`, `backward`, `turn_left`, `turn_right`, `trot`.
  - Each can run **verified** (default) or **raw** (`"verify": false`). *Verified* = move, then
    check it actually happened (camera feature-flow + ultrasonic + IMU tilt) and detect
    **stall/fall** — ~seconds per step, for autonomy. *Raw* = fire-and-return — milliseconds, for
    snappy **manual** driving. Background reflexes guard both.
- **Smart turns:** `turn_vision` — turns and **measures the real rotation from the camera** (the
  gyro under-reports ~3×); batched for speed; if it commands a turn but the view doesn't change it
  knows it's **wedged → backs up to make clearance and retries** (up to 3×), then stops and waits.
- **`face <label>`** — fast turn-to-face a landmark: pans the **head** to find the target's bearing
  (quick, no body drift), then turns the body to it via `turn_vision`.
- **`stop`** — true **emergency stop**: handled outside the action lock so it *interrupts* a running
  turn/routine immediately (≈2 ms) instead of queuing behind it.
- **Posture / expression:** `stand`, `sit`, `lie`, `look` (head), `head_center`, `bark`, `rgb`,
  `battery_led`, etc.

### Onboard "graduated" routines
These are the robust pieces pushed onto the Pi so they run network-free: `verified_move`
(stall/fall detection), `turn_vision` (measured turn + stuck-recovery), `face_landmark`
(head-scan + turn), the obstacle and fall reflexes, and emergency-stop abort.

### SLAM pipeline (separate, only when mapping)
ORB-SLAM3 (monocular), built from source at `~/ORB_SLAM3` on the Pi. Started/stopped on demand
(not part of the always-on server). See **[SLAM pipeline — detailed](#slam-pipeline--detailed)**
below for the file-by-file data flow, config, and persistence mechanism.

---

## On the MAC
Nothing here touches hardware — it all talks to the dog over the network.

### `./dog` — command-line helper (Bash)
Quick operator commands, each opens an SSH session to the Pi and curls `localhost:8080`:
`sense`, `see` (pull a frame), `perceive` (frame + sense), `scan`, `health`, `up` (reachability),
`quiet` (turn autonomy off), `approach`/`unstick` (run nav helpers).

### `cockpit/cockpit.py` — live dashboard (`http://localhost:8088`)
A web dashboard that **proxies to the dog over HTTP** (resolves `pidog.local`→IPv4 to dodge a 5 s
IPv6-timeout that used to cause lag). Shows the camera, sensors, and a SLAM map panel; lets you
**drive manually** (keyboard, snappy `verify:false` commands), records sessions, and has a
**MANUAL / AUTONOMOUS toggle** so you always know which mode is active.

### Navigation routines
See **[Navigation — exactly how it works](#navigation--exactly-how-it-works-on-the-dog)** below for
the full detail. In short: each routine is a small Python script, version-controlled here but
**`scp`-deployed to the Pi and executed there** (invoked over SSH), so its tight per-step loop
curls `localhost:8080` with no Wi-Fi hop per step.

### `rec_run.py` — run recorder
Wraps any run and saves the **onboard camera** (`/frame.jpg`) + **dog data** (`/sense`) ~1×/sec,
stitched into `dog.mp4` + `data.jsonl` under `runs/` (and mirrored to GitHub `recordings/` for
phone viewing). *(External Photo-Booth screen capture can't be grabbed from the shell sandbox, so
the room-view comes from screenshots instead.)*

### Claude (the AI) — the high-level brain
Runs on the Mac, in the loop only at the high level: looks through the dog's camera, recognizes
things, sets goals, decides what graduates to an onboard routine. It is **never** in the fast
per-step control loop.

---

## How it all talks (data flows)
- **Manual driving:** Cockpit (Mac :8088) → HTTP `/act verify:false` → `dogbrain` (:8080) → legs.
  ~20 ms round trip.
- **Autonomous nav:** Claude → SSH → a nav script *on the Pi* → tight loop of `localhost:8080`
  calls (`/detect`, `/sense`, `/act`) → returns a result. Network is hit once (to launch), not
  per step.
- **Perception:** `dogbrain` owns the camera and serves `/frame.jpg`, `/detect` (YOLO),
  `/scan` (depth), `/sense`.
- **SLAM:** `dogbrain /frame.jpg` → grabber → ORB-SLAM3 → exports → `:8081` → cockpit map panel.
- **Recording:** `rec_run.py` (Mac) samples `/frame.jpg` + `/sense` → `dog.mp4` + `data.jsonl`.

## Navigation — exactly how it works on the dog
Every navigation routine is a Python script that runs **on the Pi** and drives the body by calling
`localhost:8080` in a loop. They all compose the **same handful of primitives** and the **same
rule**: *look → decide → one small move → look again* (never blind-dead-reckon a path).

### The shared building blocks
- **Landmark bearing:** `GET /detect` → for each object, `cx` (−1 left … +1 right) and `area_frac`
  (bigger = closer). Detections are **frame-smoothed** (require the object in ≥2 of 3 frames) to
  kill YOLO flicker. A landmark's true bearing from the body = `head_yaw + cx·(half-FOV≈31°)`.
- **Distance:** `GET /sense distance_cm` (ultrasonic), **median-filtered** over several reads,
  rejecting `≤0` (no echo) and out-of-range spikes.
- **Depth profile:** pan the head across yaws and read distance at each → a "silhouette" of the
  surroundings (used to fingerprint a spot and to find the most-open direction).
- **Moves:** `forward`/`backward` (verified — with stall/fall detection), `turn_vision deg`
  (camera-measured turn that backs up if wedged), `face label` (head-scan to a landmark's bearing,
  then turn the body to it).

### The routines, file by file
- **`nav_approach.py`** — visual-servo *approach* to a landmark. `quiet()` (autonomy off) first.
  Loop: median distance + smoothed `/detect` of the target. If off-center (`|cx|>0.30`) → small
  `turn_left/right`; else `forward` one step; stop when `distance < stop_cm`. If a move comes back
  `moved:false` (stalled) → **`unstick()`**: head-scan the depth profile, pick the most-open yaw,
  turn the body that way (try the other way if that turn also stalls = boxed), then drive forward —
  *"can't go this way → turn and go where the camera can see."* Up to 3 tries, then gives up. Has
  `--quiet-only`/`--unstick-only` modes; surfaced as `./dog approach` / `./dog unstick`.
- **`find_home.py`** — locate **and face** the home landmark (the bookshelf = `book`). Up to 9
  rounds: head-scan a few yaws running `/detect`; if the bookshelf appears, turn the body toward
  the yaw it was seen at and `center_book` (servo until `cx≈0`); else `turn_vision(45°)` to the
  next sector and repeat. Result = facing the bookshelf (this fixes **orientation only**). Exits on
  fallen/stuck/aborted.
- **`record_home.py`** — capture a **home-position fingerprint**. `face("book")` first (canonical
  orientation), then head-scan `−60…+60°` recording at each yaw: the **ultrasonic distance** (→ a
  depth profile) and any detected landmark's **bearing + area + confidence**. Saves
  `home_signature.json = {fwd_dist_cm, heading_deg, depth_profile{yaw:dist}, landmarks{...}}`.
- **`return_home.py`** — return to that recorded position. Loads the signature; `face("book")`
  (orientation); then steps forward/back until the live distance matches the home
  distance-to-bookshelf (±12 cm), re-facing every couple steps; finally re-scans the depth profile
  and reports `profile_err` vs home. **Honest v1:** fixes orientation + distance; it *measures* but
  does **not** fix the lateral offset (the dog can't strafe, and one wide landmark can't pin
  left-right position).
- **`home_return.py`** — leave-and-return, open-floor version. Center on `book` → record `D_home`;
  `turn_vision(180°)` to face the open room; walk forward N steps; `turn_vision(180°)` back;
  re-acquire the bookshelf; report distance error. Battery guard uses **voltage** (the `%` glitches).
- **`navigate_return.py`** — older leave-and-return, toward-bookshelf version. Center → `D_home`;
  OUT: visual-servo approach the bookshelf to ~85 cm; `turn_vision(180°)`; BACK: forward ~same #
  steps; RE-FACE: turn until `book` centered; reports distance error + a SLAM x/z cross-check.

### What works vs. what's hard (navigation)
Working: approach-a-landmark, stall→unstick, measured turns with stuck-backup, find-and-face the
landmark, record a fingerprint. **Hard/unsolved:** returning to an *exact position* — orientation
and distance-to-one-landmark are reliable, but the **lateral axis** isn't (needs a 2nd landmark to
triangulate, scan-matching, or live SLAM). And **there is no reliable position tracking**: odometry
only counts forward/back steps (~8 cm each) and ignores the big sideways drift that turns cause, so
the dead-reckoned position is wrong after any turning.

## SLAM pipeline — detailed
ORB-SLAM3 (monocular), built from source at `~/ORB_SLAM3`. Launched by `slam_live.sh`, stopped by
`slam_stop.sh`. Three pieces, decoupled through the filesystem:

1. **Grabber — `slam/live_grab_cont.py`** (on the Pi): every ~0.07 s pulls `localhost:8080/frame.jpg`
   into `~/live/rgb/<epoch>.jpg`, pruning to the last ~40 frames. (Decouples camera I/O from SLAM.)
2. **SLAM node — `slam/mono_stream.cc`** → built as `~/ORB_SLAM3/Examples/Monocular/mono_stream`.
   Tail-follows `~/live/rgb`, feeds each new frame to `TrackMonocular`, and writes exports:
   - `~/slam_latest.txt` — one status line: `state x y z yaw cloudsize matched feats frame`.
   - `~/slam_feats.txt` — 2D keypoints (+ matched flag) for the camera overlay.
   - `~/slam_cloud.txt` — accumulated 3D map points.
   - `~/live_pose.txt` — appended pose log.
3. **Static server (`python3 -m http.server 8081`)** serves those files; the **cockpit** polls
   `:8081` and draws the top-down map + feature overlay.

**Config — `slam/pidog_mono.yaml`:** pinhole `fx=fy=266` (ESTIMATED, not properly calibrated),
320×240, 1000 ORB features. The last two lines, `System.LoadAtlasFromFile/SaveAtlasToFile:
"pidog_room"`, enable **persistence**.

**Persistence (load → merge → save):** on start it loads the saved atlas (`pidog_room.osa`); the new
session **merges** into it via place-recognition when it re-sees mapped scenery; on `SIGTERM` (from
`slam_stop.sh`) it saves the merged atlas back. This needed a **patch to ORB-SLAM3 itself**
(`slam/System.cc.patched`): wait for the LocalMapping + LoopClosing threads to finish *before*
`SaveAtlas`, otherwise the save raced the mapper and segfaulted.

**Status:** monocular SLAM tracks, builds, and persists a map; it **loses tracking during turns**
(rotation + motion blur) and **relocalizes only intermittently**. A mono-**inertial** (IMU-fused)
variant exists — `slam/mono_stream_imu.cc`, `slam/pidog_imu.yaml`, `slam/live_grab_imu*.py`,
`slam/slam_live_imu.sh`, fed by dogbrain's `/imu` stream — but its IMU init **NaN-crashes**, so it's
not used. `slam/calibrate.py` is a checkerboard intrinsic-calibration helper (the `fx=266` is still
just an estimate).

## Complete file inventory
**On the Dog (Pi):**
- `dogbrain.py` — the always-on HTTP sensorimotor server (everything in the "On the DOG" section).
- `dogbrain.service` — systemd unit that keeps `dogbrain.py` running / restarts it.
- `~/ORB_SLAM3/...` (built from source, not in this repo) + the `slam/` files below, deployed to `~/`.

**SLAM (`slam/`, mirrored here, deployed to the Pi):**
- `mono_stream.cc` — monocular SLAM node (exports pose/map). `slam_live.sh` / `slam_stop.sh` —
  start/stop. `live_grab_cont.py` — frame grabber. `pidog_mono.yaml` — camera/ORB/atlas config.
  `System.cc.patched` — the atlas-save fix (mirror of the patched ORB-SLAM3 source).
- IMU-fused (unused, NaN-crashes): `mono_stream_imu.cc`, `pidog_imu.yaml`, `live_grab_imu.py`,
  `live_grab_imu_rec.py`, `slam_live_imu.sh`. `calibrate.py` — checkerboard intrinsics. `README.md`.

**On the Mac:**
- `dog` — CLI helper (Bash). `cockpit/cockpit.py` — dashboard server (:8088).
- Nav routines (deployed to Pi, run there): `nav_approach.py`, `find_home.py`, `record_home.py`,
  `return_home.py`, `home_return.py`, `navigate_return.py`. `home_signature.json` — saved home fingerprint.
- `rec_run.py` — run recorder. `runs/` — recordings (gitignored). `recordings/` — published clips.
- `sync_public_status.sh` — mirrors the docs to the public repo.

**Docs:** `ARCHITECTURE.md` (this), `OVERVIEW.md` (plain-language), `STATUS.md` (living plan),
`CLAUDE.md` (operating brief, auto-loaded), `HANDOFF.md` (older/partly-stale), `nav_test_log.md`,
`calib_test/TURN_TEST.md`, `morning_report/` (retrospective).

## Design principle
**Fast and reflexive on the Dog; smart and slow on the Mac.** The Pi handles anything that must be
quick or safe (reflexes, stall/fall, measured turns, emergency stop); the Mac/Claude handle
perception and goals. Commands cross Wi-Fi as *intentions*, not as a per-step control loop —
because a Wi-Fi hop per twitch would be too laggy to control the body.
