# Phase 3 (+ minimal Phase 5) — Navigation & Obstacle Avoidance (build plan)

> Detail doc for **Phase 3** of [PROJECT_PLAN.md](PROJECT_PLAN.md), built on the validated Phase 1/2 capture app (`phone_brain`, 0.22 % tracking drift). Goal: free-roam map an area, then have the app **plan a path on that map and drive a human "dog"** to a goal (return to start / a waypoint), reacting to new obstacles — shown in a Tesla-FSD-style map view.

## Scope decisions (made 2026-06-19)
- **v1 = single-session**: map and navigate in one run (no app restart). Cross-session relocalization is **deferred to v2** — see the Relocalization section below.
- **No loop closure** — 0.22 % drift already validated.
- **Phase 3 + a minimal slice of Phase 5 are built together** — you can't test a planner without command output + a view to follow, so they ship as one loop.
- **Autonomous exploration is a real robot capability** (corrected): the dog **decides where to go to map unknown space** (frontier-based) and moves itself there. We test it with the **human as the dog's legs** — the app issues the *same* move instructions to me that it will later issue to the robot's motors. (It is **not** free-roam-by-hand.)

## New components (on top of `phone_brain`)
- **2D occupancy grid** — project the 3D voxel map onto the floor plane → `free / occupied / unknown` cells (a robot plans in 2D).
- **Frontier detector** — find `free` cells bordering `unknown` (the edge of the explored region = candidate places to go map next).
- **Goal selector** — *explore mode:* auto-pick the best frontier; *go-to mode:* a user goal (return / waypoint).
- **A\* planner** — shortest path over `free` cells to the goal, routing *around* `occupied` cells. Each `occupied` cell is **inflated by the robot's half-width + a margin** so the path keeps clearance (never hugs a wall, leg, or corner).
- **Mover interface** — turns the path into **one move instruction at a time** (banner + AR heading arrow), auto-advanced from live pose. **Same output whether the mover is the human tester or the robot's motors.** Includes the always-on reactive stop (below).
- **Reactive safety layer** — every frame, checks the live depth **straight ahead within the robot's width**; anything closer than the stop-distance → **immediate STOP** (overrides the plan), then mark it on the map and replan. This is the real collision-prevention.
- **FSD-style map view** — top-down render: occupancy grid + ego pose + frontiers + selected goal + planned path + waypoints + current instruction.

## Obstacle avoidance — what actually stops it hitting things
No single mechanism is enough (the map is never perfect), so it's **three layers**:

1. **Plan around obstacles it already knows.** The map marks `occupied` cells where it has seen an obstacle; the route planner only walks through `free` cells and routes *around* them. Every `occupied` cell is **inflated by the robot's half-width + a margin**, so the chosen path keeps clearance and never plans into or alongside a wall/leg. → avoids *mapped* obstacles by never routing into them.
2. **Reactive STOP from live depth — the real safety net.** Independent of any plan, **every frame** it looks at the live depth **straight ahead, within its own width, out to a stop-distance (e.g. 0.4 m)**. If anything is closer than that, it **STOPS immediately**, overriding the plan. This is what actually prevents collisions, because it does **not** trust the map: it catches new/moving things (a person, a moved chair), thin things the map missed (chair legs), glass, and any map error. Fast, always-on whenever moving.
3. **Replan after a stop.** The thing that triggered the stop gets written into the map as `occupied`; the planner computes a fresh route around it. If there's no way around, it drops that goal and picks another.

**In one line:** layer 2 (live-depth stop) is what keeps it from *running into* things; layer 1 keeps it from *planning into* known things; layer 3 lets it carry on after a surprise. The reactive stop is **active from the very first movement** — nothing moves forward toward something within the stop-distance.

## Autonomous exploration — exactly what happens, from the robot powering on

Think of the floor as a grid of cells, each tagged one of three ways as the robot sees it: **OPEN** (seen, clear to walk), **BLOCKED** (seen, an obstacle/wall there), or **UNKNOWN** (not seen yet). The robot's job is to turn all the reachable UNKNOWN into OPEN or BLOCKED — i.e., see everywhere it can go.

1. **Power on.** Almost the whole grid is UNKNOWN; the robot has only seen the small patch right in front of it.
2. **Look + fill in.** As the camera sees the floor and walls ahead, those cells flip from UNKNOWN to OPEN (clear) or BLOCKED (obstacle). The seen region grows outward from the robot.
3. **Find the edges of what it has seen.** Look for cells that are OPEN but sit right next to an UNKNOWN cell. Those edges are the interesting spots — *if the robot goes there, it'll see new territory.* (The technical name for such an edge is a "frontier"; that's all the word means.)
4. **Group the edges.** Usually there are several edge stretches (an un-entered doorway, a hallway trailing off, a gap behind furniture). The robot groups each contiguous stretch into one target and marks the middle of it as a "go-here" point.
5. **Pick which edge to go to = the nearest one it can actually walk to.** For each candidate, it works out the **real walking distance** — the length of a route that stays on OPEN cells and goes *around* walls, not a straight line through them — and picks the **closest** such reachable target. (Closest-first keeps it efficient and avoids zig-zagging across the house.) *"A\* path distance" was just jargon for "shortest real walking route avoiding obstacles."*
6. **Plan the route + start walking.** It computes the step-by-step route over OPEN cells to that point and starts issuing move commands (turn / forward / stop). During the test that's the on-screen banner driving you; on the robot it's the same commands to the motors.
7. **The edge gets "eaten away."** As it walks toward that spot, the camera keeps filling in cells, so that UNKNOWN region shrinks and new edges appear further out.
8. **Arrive → re-check → repeat.** When it reaches the spot (or the map has changed a lot), it redoes steps 3–6: where are the edges now, which is the nearest reachable one, go.
9. **Stop when there's nothing left to see.** Eventually no OPEN-next-to-UNKNOWN edges remain that it can reach — meaning everywhere reachable has been seen. It announces **"exploration complete."** The space is fully mapped.

*(In one line of jargon for reference: greedy nearest-frontier exploration with A\* path planning. A smarter target-picker later = "prefer big unexplored areas, discount far ones" — but nearest-first is the right v1.)*

## Driving the human tester (= how the app will move the robot)
The phone's camera is the sensor; during the test **I am the actuator**, and the app conveys movement the same way it will to the motors:
- **Instruction banner — one command at a time:** large arrow + text — `TURN LEFT ~45°`, `WALK FORWARD`, `STOP`, `PAN PHONE TO SCAN` (fill in depth), `EXPLORATION COMPLETE`. The app watches my live pose and **auto-advances** to the next instruction once I comply.
- **AR heading arrow** on the live camera pointing toward the next path waypoint — intuitive "go this way."
- **FSD map view** for context: my pose, the target frontier, the planned path.
- Everything **logged** (pose, instruction issued, compliance, collisions) for Mac-side analysis. On the robot, the *same* command stream drives the motors instead of the banner.

## UX / buttons
- Existing: **START/STOP** (record + clean origin), **SAVE**.
- New:
  - **EXPLORE** — start/stop autonomous exploration (app drives me to map the area).
  - **SET WAYPOINT** — drop a labeled waypoint (WP1, WP2…) at the current pose.
  - **RETURN** — chooser: *Start* or a saved waypoint → plan best path → drive turn-by-turn (go-to mode).
  - **COLLISION** — operator taps when they bump something (logs pose + time).

## Increment breakdown (each = a small, independently-testable APK)
1. **2D occupancy grid + FSD map view** — voxels → top-down `free/occupied/unknown` + ego pose, live. *Test: walk, watch the grid + pose build.* (Foundation for everything.)
2. **Continual autosave** — autosave map + trajectory every few seconds while recording.
3. **Mover interface + reactive STOP** — instruction banner + AR heading arrow to a **manually-tapped goal**, *plus* the always-on live-depth stop (it won't say "forward" if something is within stop-distance). *Test: app guides me to a tapped point; step a box in front → it immediately says STOP.* (Collision safety floor from the very first movement.)
4. **A\* path planning (with safety inflation)** — plan over `free` cells to the goal, routing around `occupied` cells inflated by the robot's width; drawn on the map + driven via the mover. *Test: path keeps clearance and reaches the goal without clipping corners.*
5. **Autonomous exploration** — frontier detect → select nearest → plan → drive → repeat until done. *Test: the app walks me around to fully map a room **on its own**.*
6. **Reactive replan + collision log** — on a reactive stop, write the obstacle into the map, reroute around it (or pick another goal); COLLISION button logs any actual bump. *Test: drop a box mid-path → it stops, then reroutes around it.*
7. **Go-to mode** — SET WAYPOINT + RETURN (Start / waypoint goal) reusing the same planner + mover. *Test: navigate back to a chosen point.*

→ Start with **Increment 1**; exploration (Inc 5) falls out once the grid + mover + planner exist.

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
