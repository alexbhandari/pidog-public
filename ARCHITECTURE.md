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
ORB-SLAM3 (built from source on the Pi). A grabber pulls `/frame.jpg` into a folder; a
`mono_stream` node runs visual SLAM and writes pose + map exports; a small static server on
`:8081` serves them to the dashboard. The map is **persistent** (load → merge → save). Runs only
when started; not part of the always-on server.

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

### Navigation routines (authored on the Mac, **run on the Dog**)
These Python scripts are version-controlled in this repo but are **deployed to the Pi (`scp`) and
executed there** (invoked over SSH). Running on the Pi means their tight per-step loop curls
`localhost:8080` with no Wi-Fi hop per step:
- `nav_approach.py` — visual-servo approach to a landmark (+ `unstick` recovery).
- `find_home.py` — sweep + detect + face the bookshelf (find the home *landmark/orientation*).
- `record_home.py` — capture a **home-position fingerprint** (depth profile + landmark bearings).
- `return_home.py` — face the landmark + match the home distance, report position match.
- `home_return.py` / `navigate_return.py` — leave-and-return experiments.

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

## Design principle
**Fast and reflexive on the Dog; smart and slow on the Mac.** The Pi handles anything that must be
quick or safe (reflexes, stall/fall, measured turns, emergency stop); the Mac/Claude handle
perception and goals. Commands cross Wi-Fi as *intentions*, not as a per-step control loop —
because a Wi-Fi hop per twitch would be too laggy to control the body.
