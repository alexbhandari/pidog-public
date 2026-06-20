# Phase 1 — Unified ARCore capture app (build details)

> Detail doc for **Phase 1** of [PROJECT_PLAN.md](PROJECT_PLAN.md). Goal: one Android app that, in a single ARCore session, produces **localization (pose) + dense depth + a persistent map** and logs them so we can verify on the Mac.

## Starting point

**Fork Google's `raw_depth_java` sample** (from `google-ar/arcore-android-sdk`). It already provides ~60% of Phase 1: ARCore session lifecycle, permissions, GL rendering, **camera pose**, **Raw Depth API**, and the depth-image + intrinsics → camera-space → world-space reprojection. We strip its demo UI and add persistence, logging, and export.

**Language: Java first** (the sample is Java) to maximize reuse; Kotlin can be added incrementally later. Build in **Android Studio**; run on the **S22** (ARCore + Depth API supported).

## What we pull from each repo

| Repo | Take | Ignore |
|---|---|---|
| **google-ar/arcore-android-sdk** (`raw_depth_java`, `hello_ar_java`) | App skeleton — session, permissions, GLSurfaceView, **pose**, **Raw Depth API**, depth→world reprojection, point-cloud render | Demo UI/buttons |
| **nadiawangberg/phone-slam** | Depth **confidence filtering** + point-cloud accumulation approach | Research scaffolding |
| **eschoenawa/Urban-Scanner** | Persistent accumulation + **PLY export** patterns, cleaner app structure | Geospatial/GPS anchoring |
| **introlab/rtabmap** | Reference only — loop-closure / occupancy-grid concepts + the **offline desktop tool** (see loop-closure section) | Don't fork/embed the C++ now |
| **lvonasek/3DLiveScanner** | Reference only — export-format + "dataset" (frames+poses) ideas | The C++/JNI engine |

**Net new code we write:** persistent voxel/occupancy map + pose-trajectory logger + file exporter. Modest.

## Components to build (new files on top of the fork)

- `MapAccumulator` — fuse `(pose, confidence-filtered depth points)` into a deduplicated voxel/occupancy grid (the validated version of `runmap.py`'s `map_voxels`).
- `PoseLogger` — append per-frame 6DoF pose (+ timestamp) to `trajectory.json`/CSV.
- `PlyExporter` — write the accumulated map to `map.ply` (point cloud; mesh later).
- Wire into the sample's frame loop; add a minimal on-screen Start/Stop + live map view.

## Data flow to the Mac

**No automatic phone→Mac sharing, and Phase 1 doesn't need it.**
1. **Primary view = on-phone** (the app renders the map + pose live as you walk).
2. **Verification = on Mac** via the pipeline we already have: app writes files to phone storage → **`adb pull`** → existing `tools/` scripts visualize/measure.

On-device outputs (under app files dir, pulled with adb):
- `map.ply` — accumulated point cloud / occupancy
- `trajectory.json` — per-frame poses (+ timestamps)
- `depth/` — (optional) per-frame depth for offline reprocessing

*Deferred:* live WiFi streaming to the Mac for real-time monitoring — a Phase-2 nicety, not needed to prove Phase 1.

## Loop closure — deferred, but its impact measured for ~free

Real loop closure (place recognition + pose-graph optimization) is **not** a quick in-app add — it means adopting RTAB-Map or writing a mini-SLAM backend. **So Phase 1 does not build it.** Instead we measure whether it's needed:

1. Phase 1 logs **raw ARCore pose** (free — already available).
2. Walk a **deliberate closed loop**, returning to a marked floor spot.
3. Quantify drift (computed by `tools/` from `trajectory.json`):
   - **End-to-start gap** (m)
   - **Drift rate** = gap ÷ path length (%)
   - **Repeat-pass misalignment** (overlay the two passes of the same corridor)
4. **Free baseline:** we already have RTAB-Map's **loop-closed** trajectory on the same physical route → raw-ARCore drift vs RTAB-Map-corrected = the **quantified value of loop closure, with no loop-closure code written**.
5. If it matters, the still-cheap next step is to run our recorded frames+poses **through RTAB-Map offline on the Mac** before any in-app integration.

**Decision rule:** end-gap under ~1–2% of path length on a house loop → skip loop closure (ARCore's built-in COM correction is enough). Larger → integrate RTAB-Map at Phase 3, quantified not guessed.

## Acceptance criteria (Phase 1 done)

- App builds + runs on the S22; live map builds as you walk.
- Pulled `map.ply` is metric and shows real structure (walls, furniture, obstacles visible).
- `trajectory.json` gives a clean pose path; end-to-start gap measurable on a loop.
- Confidence-filtered depth removes obvious garbage points.

## Build / run

- Open the app module in **Android Studio**; ARCore dependency via Gradle (`com.google.ar:core`).
- Deploy to S22 over the existing **wireless adb** link; grant Camera permission.
- Pull outputs: `adb pull <app-files-dir>/… scans/phase1/`.
- Convenience: `./phone_brain/dev.sh build|install|launch|pull`.

## Build environment (required on this Mac — CLI, no Android Studio)

The toolchain was installed by direct download (no Homebrew/Xcode), so a few one-time steps are **required** or the build fails:
- **`JAVA_HOME` must point to `tools/jdk`** (Temurin 17). With the system Java 8, AGP 8.4 fails dependency resolution (`compatible with Java 8 … needed Java 11`).
- **Ad-hoc code-sign the native binaries** or macOS Gatekeeper SIGKILLs them when Gradle forks them (`Failed to exec spawn helper: signal 9` / `AAPT2 Daemon startup failed`):
  ```bash
  codesign --force --sign - tools/jdk/Contents/Home/lib/jspawnhelper
  find ~/.gradle tools/android-sdk -name aapt2 -type f -exec codesign --force --sign - {} \;
  ```
- `dev.sh` sets `JAVA_HOME`/`ANDROID_HOME` for you; re-run the `codesign` lines only if the JDK/SDK is reinstalled.

> Process note: always stream build output to a tailable log (`> build.log 2>&1`) and watch it live — buffered output once made normal builds look like hangs.

## Result (Phase 1 done)

Closed-loop walk on the S22: **35.5 m path, 7.8 cm end-to-start gap, 0.22 % drift**, 89k-voxel dense map. Conclusion: **ARCore VIO is accurate enough at this scale → no loop closure needed.**
