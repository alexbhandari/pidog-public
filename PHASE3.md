# Phase 3 (+ minimal Phase 5) — Navigation & Obstacle Avoidance (build plan)

> Detail doc for **Phase 3** of [PROJECT_PLAN.md](PROJECT_PLAN.md), built on the validated Phase 1/2 capture app (`phone_brain`, 0.22 % tracking drift). Goal: free-roam map an area, then have the app **plan a path on that map and drive a human "dog"** to a goal (return to start / a waypoint), reacting to new obstacles — shown in a Tesla-FSD-style map view.

## Scope decisions (made 2026-06-19)
- **v1 = single-session**: map and navigate in one run (no app restart). Cross-session relocalization is **deferred to v2** — see the Relocalization section below.
- **No loop closure** — 0.22 % drift already validated.
- **Phase 3 + a minimal slice of Phase 5 are built together** — you can't test a planner without command output + a view to follow, so they ship as one loop.
- **"Autonomous explore" = free-roam mapping.** You walk around naturally; the map builds + autosaves. The app does **not** direct you during mapping (frontier-style "go map that gap" is a *robot* behavior, deferred). The autonomy we build/test is **navigation back**.

## New components (on top of `phone_brain`)
- **2D occupancy grid** — project the 3D voxel map onto the floor plane → `free / occupied / unknown` cells (a robot plans in 2D).
- **A\* planner** — shortest path on the grid from current pose to a goal.
- **Reactive layer** — live depth in front → detect a *new* obstacle not on the map → stop / replan.
- **Command generator** — path → `stop / forward / turn L / turn R / back up`.
- **FSD-style map view** — top-down render: occupancy grid + ego pose + planned path + waypoints + current command.
- **Waypoint store** + **RETURN chooser**.

## UX / buttons
- Existing: **START/STOP** (record + clean origin), **SAVE**.
- New:
  - **SET WAYPOINT** — drop a labeled waypoint (WP1, WP2…) at the current pose.
  - **RETURN** — chooser: *Start* or a saved waypoint → plan best path → drive turn-by-turn.
  - **COLLISION** — operator taps when they bump something (logs pose + time).

## Increment breakdown (each = a small, independently-testable APK)
1. **2D occupancy map view** — voxels → top-down grid, rendered live with ego pose. *Test: walk, watch free/occupied/unknown + pose build.* (FSD-view foundation; de-risks everything.)
2. **Continual autosave** — autosave map + trajectory every few seconds while recording.
3. **Waypoints** — SET WAYPOINT + render them on the map.
4. **Path planning (draw only)** — RETURN → pick goal → A\* path drawn on the grid. *Test: path looks sane / routes around obstacles.*
5. **Turn-by-turn guidance** — path → stop/fwd/turn commands shown one at a time. *Test: follow them to the goal as the "dog".*
6. **Reactive avoidance** — live depth catches a new obstacle → stop/replan, + COLLISION button. *Test: drop a box in the path, see it stop/reroute.*

→ Start with **Increment 1**.

## How we test (human-as-dog)
Operator holds the phone, follows the on-screen commands, taps **COLLISION** on any bump. The app logs poses / commands / events; we pull and analyze on the Mac (reuse `tools/`). Acceptance: reaches the goal following commands, replans around a new obstacle, collisions logged, the path on the map view matches reality.

---

## Deferred to v2 — cross-session relocalization (the "wake up and know where I am" problem)

**Why deferred:** navigating a *stored* map across app restarts requires the phone to localize into a map built earlier. But **every ARCore session starts at its own origin**, so a saved map doesn't auto-align with a fresh session. v1 sidesteps this by staying single-session.

**Findings so far (2026-06-19):**
- **RTAB-Map localization mode** (on-device, offline) test on the S22: relocalized **reliably** (logcat: 107 inlier points / 99 % ratio — a confident place match) **but** with a **~30 cm (≈1 ft) offset** and a **slow initial lock**, and the offset didn't visibly shrink while moving. Good enough for coarse room-level navigation; risky for tight obstacle clearance / docking.

**Planned test — Cloud Anchors in our app + a quantitative harness (effort: medium):**
- **Method (rigorous, no eyeballing):** mark a fixed start spot. Session 1 hosts an anchor **at that spot** + saves the map. Session 2 returns the phone to the **same spot**, resolves the anchor, and **logs the resolved-pose deviation** — translation magnitude = **position error (cm)**, rotation = **heading error (deg)**. Position error is robust to small hand-held wobble, so a floor mark suffices; a phone jig makes heading repeatable. A visual map overlay (like RTAB-Map's) is the sanity check.
- **Why this harness matters:** it's **backend-agnostic** — same measurement scores Cloud Anchors, a future on-device ORB relocalizer, or a re-test of RTAB-Map. Build once, reuse.
- **Pieces:** GCP ARCore Cloud Anchor API key (~10 min, user) · host/resolve wiring · resolved-pose error logging · transformed map overlay render. Standard Cloud Anchors persist 24 h (fine for same-day tests; multi-day needs keyless auth — defer).
- **Robot-friendly fallbacks if relocalization stays weak:** **dock-start** (robot boots from the same physical spot each time → repeatable origin, no vision needed) or an **Augmented Image fiducial** (printed marker = known reference).

**Decision pending:** Cloud Anchors vs. on-device ORB vs. adopting RTAB-Map for the localization backend — to be made after the Cloud Anchors numbers are in.
