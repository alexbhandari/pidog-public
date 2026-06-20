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
9. **Before stopping, look around.** The camera only sees a **forward cone (~65°)**, so "no edges *in front*" does **not** mean done — there may be unmapped space beside or behind it. So when it runs out of edges ahead, it first **turns in place to sweep the camera around** (a look-around scan). Only if a full look-around still finds **no reachable OPEN-next-to-UNKNOWN edge** does it announce **"exploration complete."** This stops it from quitting with unseen space behind it. (It also does a quick scan on *arriving* at each target, to map the newly revealed area before picking the next one.)

*(In one line of jargon for reference: greedy nearest-frontier exploration with A\* path planning. A smarter target-picker later = "prefer big unexplored areas, discount far ones" — but nearest-first is the right v1.)*

## Robot motion model — it is not a free-flying dot
Two physical facts about the PiDog change the planning; the plan must respect them.

**A. Limited field of view → it must look around** (covered in exploration step 9): the camera sees only a forward cone, so the robot **scans on arrival** and does a **full in-place look-around before declaring "complete."** It never quits just because nothing is in front of it.

**B. Non-holonomic motion — shallow turns, can't spin on the spot.** The dog moves forward with a limited steering angle (a **minimum turning radius**), like a car, not a tank. A plain grid route with 90° corners is **not drivable**. v1 approach:
- Grid A\* gives a rough route → **smooth it into gentle arcs no tighter than the dog's turning radius**.
- Commands are **curve-based**, not pivot-based: `FORWARD`, `BEAR LEFT` / `BEAR RIGHT` (gentle turn *while* moving), `STOP`, `BACK UP`.
- If a spot needs a turn sharper than the radius → insert a maneuver (tightest arc, or back-up-and-reposition — see C).
- **Calibration:** the turning radius + speeds come from a one-time **on-dog** calibration — see **Motion calibration** below (including the placeholder assumed for the human test).
- *(Later: a proper kinematic planner — Hybrid A\* / Reeds–Shepp — natively emits drivable paths, including reverse.)*

**C. Backing up — when and how.** The forward-only mover gets stuck in three cases, all handled by a **reverse maneuver**:
- **Dead-end / too-tight turn** — the only exit needs a turn tighter than the radius.
- **Blocked ahead with no room to curve around** (after a reactive stop).
- **Reversing direction** in a narrow space (back-up + arc, like a 3-point turn).

Handling: a *stuck* handler — when no drivable forward path makes progress, issue **`BACK UP`**, but **only over cells already mapped OPEN** (the camera can't see behind, so it reverses only into space it just drove through), then re-orient and replan. Reverse is short and slow; if even backing up opens no route, it abandons that target and picks another.

## Motion calibration (measure the dog's real motion)
**What it measures** (the planner needs all of these): forward speed; **min turning radius LEFT *and* RIGHT, measured separately** (they should mirror — same radius, opposite direction — but we measure both to *confirm*; if asymmetric, store per-side); turn rate; **reverse speed**; and whether it can steer while reversing.

**Steps** — each step: issue **one fixed command for a set time T**, read the **ARCore pose before & after** (the on-dog phone is the odometer), repeat ×3 and average:
1. **Forward speed** — `FORWARD` straight for T → distance ÷ T (m/s).
2. **Turn radius LEFT** — full-left steer + forward for T → it traces an arc; from Δposition + Δheading, `R_left = arc_length ÷ heading_change(rad)`; also record turn rate (°/s).
3. **Turn radius RIGHT** — full-right steer + forward for T → `R_right`. (Expect `R_left ≈ R_right`.)
4. **Reverse speed** — `BACK UP` straight for T → distance ÷ T.
5. **Reverse steering (optional)** — if it can steer while reversing, repeat 2–3 in reverse → reverse radius; else **reverse = straight-only**.

→ Writes a config: `{forward_speed, reverse_speed, turn_radius_left, turn_radius_right, turn_rate, reverse_steerable}`.

**When it runs:** on the **real dog, at Phase 6** — you can't measure a dog that doesn't exist yet.

**How the human test handles it (the assumption):** the planner reads these params from a **config file**, so the human test fills it with **placeholders** and swapping in calibrated numbers later is a config edit — *no code change*:
- **Assumed min turning radius `R = 0.6 m`, symmetric (left = right).** *(A guess for a small quadruped's shallow turn — give me a better number if you have one and I'll use it.)*
- **Reverse = straight-only.**
- **Forward speed: not needed for the human test** — commands auto-advance from live pose, so they adapt to my walking speed.

During the test I follow curve commands sized to `R = 0.6 m` (the AR arc shows the curve), so the planner is exercised under a dog-like constraint *before* the real numbers exist. Phase 6 then runs the calibration and overwrites the placeholders.

## Driving the human tester (= how the app will move the robot)
The phone's camera is the sensor; during the test **I am the actuator**, and the app conveys movement the same way it will to the motors:
- **Instruction banner — one command at a time, curve-based** (matches the dog's shallow turns, not pivots): `FORWARD`, `BEAR LEFT` / `BEAR RIGHT` (gentle turn while walking), `STOP`, `BACK UP`, `SCAN` (sweep to look around), `COMPLETE`. The app watches my live pose and **auto-advances** once I comply — and I make the *same* shallow turns the dog will, so the planning is tested under the real motion constraint.
- **AR heading arrow** on the live camera pointing toward the next path waypoint — intuitive "go this way."
- **FSD map view** for context: my pose, the target frontier, the planned path.
- Everything **logged** (pose, instruction issued, compliance, collisions) for Mac-side analysis. On the robot, the *same* command stream drives the motors instead of the banner.

### Interface mockup (what I'll be looking at)
Phone held landscape. Live camera fills the screen; the big banner = the one thing to do *now*; the AR arc on the floor shows the curve to follow; the corner minimap gives context.

```
┌──────────────────────────────────────────────────────────────┐
│ ● REC   voxels 12,480   poses 920            mode: EXPLORE     │
│                                                                │
│                 ╔════════════════════════╗                     │
│                 ║   ⤺  BEAR LEFT          ║   ← do this NOW     │
│                 ║   keep walking, gentle  ║                     │
│                 ╚════════════════════════╝                     │
│                                                                │
│                    ( live camera view )                        │
│                          ╭╮                                    │
│                         ╱  ╲   ← AR arc on the floor:           │
│                        ╱        the path to walk                │
│   ┌─────────────┐                                              │
│   │  MINIMAP    │   ▓ wall/blocked   · open   ? unknown        │
│   │ ▓▓▓▓▓▓▓▓▓   │   ●▸ you (▸ = heading)                       │
│   │ ·· ●▸· ·?   │   ✕ target frontier   ┄ planned path         │
│   │ ·· ┄┄ ✕?    │                                              │
│   │ ▓▓▓· ·?? ?  │                                              │
│   └─────────────┘                                              │
│  [EXPLORE] [WAYPOINT] [RETURN] [⚠ COLLISION]        [■ STOP]    │
└──────────────────────────────────────────────────────────────┘
```

**Minimap = heading-up, like car GPS / FSD:** your icon stays fixed pointing **up**, and the whole map **rotates around you** so "up on screen" is always "forward in real life." It always shows the **planned route** (`┄`) from you to the target, plus open/blocked/unknown and frontiers. (Toggle to north-up is a later nicety.)

**The banner is the whole UX** — you only ever do what it says. Its full vocabulary:

| Banner | What you (and later the dog) do |
|---|---|
| `▲ FORWARD` | walk straight ahead |
| `⤺ BEAR LEFT` / `⤻ BEAR RIGHT` | gentle turn **while still walking** (shallow, ~0.6 m radius arc — *not* a pivot) |
| `■ STOP` | stop immediately (also fires on the reactive depth-stop) |
| `▼ BACK UP` | step backward slowly (only over already-seen floor) |
| `↻ SCAN` | turn slowly in place to sweep the camera and look around |
| `✓ COMPLETE` | exploration / arrival done |

It shows **one** banner at a time and **auto-advances** the moment your live pose shows you've complied — so it always reflects the next move, and the AR arc + minimap keep you oriented.

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
3. **Mover interface + reactive STOP** — **curve-based** banner (`FORWARD` / `BEAR L` / `BEAR R` / `STOP` / `BACK UP` / `SCAN`) + AR heading arrow to a **manually-tapped goal**, *plus* the always-on live-depth stop. *Test: app guides me with shallow-turn commands; step a box in front → it immediately says STOP.* (Includes a quick one-time **turning-radius + speed calibration**. Collision safety floor from the first movement.)
4. **A\* path planning (drivable, with safety inflation)** — plan over `free` cells, inflate `occupied` by robot width, then **smooth the route into arcs no tighter than the turning radius**. *Test: path keeps clearance, is followable with shallow turns, reaches the goal without clipping corners.*
5. **Autonomous exploration (with look-around)** — frontier detect → select nearest → plan → drive → **scan on arrival** → repeat; **full look-around before "complete."** *Test: the app walks me around to fully map a room **on its own**, and doesn't quit with space unseen behind me.*
6. **Reactive replan + back-up handler + collision log** — on a reactive stop, write the obstacle to the map and reroute; if stuck (dead-end / too-tight turn / no forward escape), issue **`BACK UP`** over already-seen `free` cells, re-orient, replan; COLLISION button logs any actual bump. *Test: box mid-path → stop+reroute; dead-end → backs up and turns around.*
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
