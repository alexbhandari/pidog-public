# Pidog Spatial-AI Project Plan

**Goal:** A Samsung Galaxy S22 acts as the "brain + sensors" for a PiDog robot (S22 → Raspberry Pi over USB). The phone maps the house, localizes within it, detects obstacles and depth, plans routes, avoids obstacles, and (later) recognizes/classifies objects — to autonomously navigate, patrol, find things, and interact.

**Strategy:** Build the whole stack on the **phone first**, validating each capability with a human standing in for the dog (follow on-screen commands, log collisions). Only deploy to the PiDog once it works on the phone.

**Device facts:** S22 = Snapdragon/Exynos, **no LiDAR/ToF** → depth comes from ARCore depth-from-motion. ARCore is well supported on this device. iPhone-15 RTAB-Map experience (no LiDAR either) set the quality bar; tracking is expected to transfer.

---

## Target system architecture

One ARCore app on the phone is the brain: it reads **pose + depth** straight from the ARCore SDK (the two things RTAB-Map and 3D Live Scanner each withhold), builds a persistent map, plans/avoids, and emits movement commands — first to a human stand-in, later to the PiDog.

Blocks are **grouped by the phase that builds them** and tagged with **status** (✅/🔄/⬜/⚠️). Color = neural-net usage: **green = no NN** (classical geometry/search), **red = NN**. Yellow notes give each net's **inputs → outputs → why**. Note that **localization and mapping use no NN** — the *only* net in core sensing is the Depth API, and it's optional (skip it and the map is just sparse).

```mermaid
flowchart TD
    CAM["📷 Camera + IMU<br/>✅ sensor input (always on)"]

    subgraph P1["Phase 1 — Unified ARCore app · ⬜"]
        ARCORE["ARCore session (COM)<br/>⬜ not started"]
        POSE["Localization · 6DoF pose (VIO)<br/>⬜ · NO NN — classical visual-inertial<br/>odometry; camera corrects IMU drift"]
        DEPTH["Dense depth · ARCore Depth API<br/>⬜ · 🧠 NEURAL NET (depth-from-motion)<br/>optional — skip = sparse map, no NN"]
        MAP["Persistent occupancy / voxel map<br/>⬜ · NO NN — geometric accumulation<br/>(fuses pose + depth into a world map)"]
    end

    subgraph P4["Phase 4 — Perception (optional) · ⚠️"]
        ML["ML Kit / MediaPipe / TFLite<br/>object detection + segmentation<br/>⚠️ decision needed · 🧠 NEURAL NET"]
    end

    subgraph P3["Phase 3 — Navigation · ⬜"]
        PLAN["Route planning +<br/>obstacle avoidance<br/>⬜ · NO NN — search + geometry"]
        CMD["Commands: stop / forward /<br/>turn L / R / back<br/>⬜ not started"]
    end

    subgraph P5["Phase 5 — Human-in-loop test · ⬜"]
        VIEW["FSD-style map view<br/>+ collision log<br/>⬜ not started"]
        HUMAN["🧍 Human stand-in<br/>⬜ not started"]
    end

    subgraph P6["Phase 6 — Robot · ⬜"]
        PI["Raspberry Pi<br/>⬜ not started"]
        DOG["🐕 PiDog motors<br/>⬜ not started"]
    end

    CAM --> ARCORE
    ARCORE --> POSE
    ARCORE --> DEPTH
    POSE -->|"where am I"| MAP
    DEPTH -->|"what's around (metric)"| MAP
    ARCORE -.->|"frames"| ML
    ML -.->|"semantic labels"| MAP
    MAP --> PLAN
    PLAN --> CMD
    CMD --> VIEW
    VIEW -.->|"human follows"| HUMAN
    CMD -->|"USB"| PI
    PI --> DOG

    NN1["🧠 Neural net — ARCore Depth API (the ONLY NN in core sensing)<br/>IN: live camera frames + device motion (VIO pose)<br/>OUT: per-pixel metric depth map + confidence<br/>WHY: S22 has no LiDAR, so a CNN infers depth from motion<br/>parallax → dense metric obstacles. Localization/mapping don't use it."]
    NN2["🧠 Neural net — ML Kit / MediaPipe / TFLite<br/>IN: RGB camera frames<br/>OUT: object boxes + class labels (detection),<br/>and/or per-pixel class masks (segmentation)<br/>WHY: tells the robot WHAT things are — finding,<br/>interacting, patrolling. ARCore can't."]
    DEPTH -.-> NN1
    ML -.-> NN2

    classDef note fill:#ffd,stroke:#cc0,color:#000;
    classDef nonn fill:#d6f5d6,stroke:#3a3,color:#000;
    classDef nn fill:#f8d2d2,stroke:#c33,color:#000;
    class NN1,NN2 note;
    class POSE,MAP,PLAN nonn;
    class DEPTH,ML nn;
```

---

## Status legend
✅ done · 🔄 in progress · ⬜ not started · ⚠️ blocked/decision needed

---

## Roadmap at a glance

```mermaid
flowchart LR
    P0["Phase 0<br/>Sensing validation"]:::done --> P1["Phase 1<br/>Unified ARCore app<br/>pose + depth + map"]
    P1 --> P2["Phase 2<br/>On-phone test<br/>+ visualize"]
    P2 --> P3["Phase 3<br/>Planning +<br/>obstacle avoidance"]
    P3 --> P5["Phase 5<br/>FSD-style nav test<br/>human-in-loop"]
    P5 --> P6["Phase 6<br/>Deploy to PiDog"]
    P3 -.-> P4{"Phase 4<br/>Add classification<br/>/ segmentation?"}
    P4 -.-> P5
    classDef done fill:#bbf,stroke:#27a,color:#000;
```

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

## Phase 1 — Unified capture app (🔄)

📄 **Build details: [PHASE1.md](PHASE1.md)** — starting point (fork Google `raw_depth_java`), per-repo sourcing, data-to-Mac flow, loop-closure measurement.

Build one ARCore Android app (Java-first, forked from the `raw_depth_java` sample) that produces, in a single session, **all** the data the robot needs:
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
