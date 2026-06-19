# Pidog Spatial-AI Project Plan

**Goal:** A Samsung Galaxy S22 acts as the "brain + sensors" for a PiDog robot (S22 → Raspberry Pi over USB). The phone maps the house, localizes within it, detects obstacles and depth, plans routes, avoids obstacles, and (later) recognizes/classifies objects — to autonomously navigate, patrol, find things, and interact.

**Strategy:** Build the whole stack on the **phone first**, validating each capability with a human standing in for the dog (follow on-screen commands, log collisions). Only deploy to the PiDog once it works on the phone.

**Device facts:** S22 = Snapdragon/Exynos, **no LiDAR/ToF** → depth comes from ARCore depth-from-motion. ARCore is well supported on this device. iPhone-15 RTAB-Map experience (no LiDAR either) set the quality bar; tracking is expected to transfer.

---

## Status legend
✅ done · 🔄 in progress · ⬜ not started · ⚠️ blocked/decision needed

---

## Phase 0 — Sensing validation (✅ DONE, 2026-06-19)

**Question:** Can the S22 + ARCore actually track, map, and produce depth well enough? Which existing app, if any, gives localization + loop closure + dense depth + obstacles?

**Method:** Installed three open-source apps via wireless adb. Walked the **same route** capturing with RTAB-Map (×3) and 3D Live Scanner (×2). Pulled the raw files to Mac and reverse-engineered them (`tools/` scripts).

### Results

| Capture | Nodes/verts | Localization | Loop closure | Dense depth saved | Obstacles/occupancy | Trajectory accessible |
|---|---|---|---|---|---|---|
| RTAB-Map `083419` | 104 nodes | ✅ | 7 global | ❌ `depth=0` | ❌ `obstacle_cells=0` | ✅ (pose graph) |
| RTAB-Map `084330` | 90 nodes | ✅ | 5 global | ❌ | ❌ | ✅ |
| RTAB-Map `091312` (route) | 262 nodes | ✅ | **9 global** | ❌ `depth=0` | ❌ | ✅ |
| 3D Live Scanner `084038` | 245,922 verts | ❌ not exposed | ❌ none | ✅ dense mesh | (implicit in mesh) | ❌ paywalled |
| 3D Live Scanner `092541` (route) | 187,306 verts / 270,076 faces | ❌ | ❌ | ✅ dense mesh | (implicit) | ❌ |

### Key conclusions
1. **ARCore tracking on the S22 is solid.** RTAB-Map (which runs *on* ARCore) held tracking over 14 m with no loss and produced 5–9 real appearance-based **loop closures** — the globally-consistent localization we need. ARCore also has its own internal drift correction ("Concurrent Odometry and Mapping"), which is why 3DLS tracking felt good too.
2. **No single app gives us everything:**
   - **RTAB-Map** = excellent **localization + loop closure**, but **does not persist depth or occupancy** in these captures (only RGB + sparse feature scans + a decimated assembled cloud). Sparse map = the user's stated concern, confirmed.
   - **3D Live Scanner** = **dense depth-derived mesh**, but **poses/trajectory are paywalled** (the "Capture a dataset" feature) and there's no loop closure.
3. **The thing that withholds data from us — pose (3DLS) and depth (RTAB-Map) — ARCore exposes BOTH directly** via its SDK (camera pose + Depth API) in one session, with no paywall and full control.

### ⚠️ Architecture decision (resolved by Phase 0)
**Build our own ARCore app** that reads **pose + Depth API together**, rather than (a) modifying RTAB-Map's 5,000-line C++ to save depth, or (b) paying/hacking 3DLS for poses. Pull *techniques/reference* from both, but the app is ours.
- **Localization + depth + obstacles**: directly from ARCore in our app (Path A).
- **Loop closure / global map consistency**: start without it (ARCore's internal correction is decent at room scale); add a lightweight loop-closure step or integrate RTAB-Map as a *library* later **if** drift proves to matter at house scale (Phase 3 decision).

### Artifacts (on disk)
- `scans/rtabmap/*.db` (raw), `*_cloud.ply` / `*_mesh.ply` (extracted, open in CloudCompare/MeshLab), `*_preview.png`, `trajectories.png`
- `scans/3dlivescanner/route_092541/*.obj` (dense textured mesh)
- `tools/` — extraction/analysis scripts; `tools/platform-tools/adb` (wireless link to phone)

---

## Phase 1 — Unified capture app (⬜)

Build one ARCore Android app (native Kotlin) that produces, in a single session, **all** the data the robot needs:
- [ ] ARCore session: 6DoF **pose** stream (localization)
- [ ] **Depth API**: per-frame metric depth (obstacles + depth)
- [ ] Accumulate depth+pose into a **persistent occupancy/voxel map** (the `map_voxels` idea from the Mac prototype `runmap.py`, now on correct metric data)
- [ ] Save/export: map (voxel grid + cloud), full **pose trajectory**, per-frame depth — in formats we can pull and inspect (PLY/JSON)
- [ ] Reference implementations to mine: Google `arcore-android-sdk` `raw_depth` sample, `nadiawangberg/phone-slam` (raw-depth point cloud + confidence filtering)
- **Decision to confirm:** lightweight own-mapping now vs. linking `librtabmap` for loop closure from the start. *Lean: own-mapping first.*

## Phase 2 — On-phone test + visualization (⬜)

Confirm Phase 1 produces correct localization, obstacles, and depth by **walking the same route** used in Phase 0.
- [ ] Live on-phone view of the map building + current pose
- [ ] Pull data to Mac; visualize/verify trajectory, occupancy map, depth (reuse `tools/` viz)
- [ ] Acceptance: map is metric & consistent; obstacles (chair legs, table edges, walls) clearly present; pose stable over the loop

## Phase 3 — Route planning + obstacle avoidance (⬜)

- [ ] **Initial run (navigate-while-mapping):** plan/avoid using live depth as the map is built (SLAM-style exploration)
- [ ] **Revisit (navigate-on-stored-map):** localize into the saved map, navigate easily from it, **refine** it as you go
- [ ] **Dynamic obstacles:** detect & react to NEW obstacles not in the stored map (replan locally)
- [ ] **Decision:** does house-scale drift now require real loop closure? If yes → add it / integrate RTAB-Map lib here.

## Phase 4 — Object classification & segmentation (⚠️ decision)

Needed later for finding things, interacting, patrolling, and "knowing what is what" while navigating.
- **Note:** ARCore does **not** do general indoor object classification/segmentation. Its *Scene Semantics* API is outdoor-oriented (sky/road/building). For indoor we'd add **ML Kit** (object detection, image labeling), **MediaPipe**, or a **TFLite** segmentation model — all native Android, composable with the ARCore app.
- [ ] Decide scope: detection/labeling (cheap) vs. full semantic segmentation (heavier)
- [ ] Decide build: off-the-shelf models vs. fine-tuning (defer training unless clearly needed)
- [ ] Fuse labels into the map (semantic voxels / object anchors)

## Phase 5 — Navigation test ("FSD-style" human-in-the-loop) (⬜)

Turn it into a navigation app the human operator follows as the "dog."
- [ ] Rendered map view (Tesla-FSD-style): map + ego pose + planned path + detected obstacles (+ object labels if Phase 4)
- [ ] Command output: **Stop / Forward / Turn L / Turn R / Back up**
- [ ] **Collision-log button** (operator taps when they hit something) + run logging (pose, command, events)
- [ ] Run the route(s); analyze logs for collisions/failures
- [ ] Iterate: tune, debug, possibly fine-tune/train models (avoid training if disproportionate)

## Phase 6 — Deploy to PiDog (⬜)

- [ ] S22 → Pi over USB; Pi relays motor commands
- [ ] Integrate brain (phone) ↔ body (Pi/PiDog)
- [ ] Re-run navigation/collision tests on the real robot; iterate

---

## Open decisions tracker
- ⚠️ **P1:** own lightweight mapping vs. `librtabmap` integration (lean: own first)
- ⚠️ **P3:** when/whether to add real loop closure (trigger: measured house-scale drift)
- ⚠️ **P4:** classification scope + off-the-shelf vs. trained models
- **Control location:** phone-brain vs. phone-maps/Pi-plans split (revisit at Phase 6)

## Changelog
- **2026-06-19:** Created plan. Completed Phase 0 (sensing validation) — see results above. Resolved core architecture decision: build our own ARCore app reading pose + depth, since neither RTAB-Map (no saved depth/obstacles) nor 3D Live Scanner (no accessible poses) provides the full set.
