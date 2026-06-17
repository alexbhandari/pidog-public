# PiDog — a robot dog learning to find its way around (plain-language overview)

*This is the friendly version. For the technical details, see [STATUS.md](STATUS.md).*
Last updated: **2026-06-17**.

## What this project is
A small **robot dog** (a SunFounder "PiDog" with a little computer and a camera in its nose)
is learning to move around a room **using only its own eyes** — the way a real dog or person
does. No cameras on the ceiling, no map handed to it. An AI (Claude) acts as its "remote
brain": it looks through the dog's camera, decides what to do, and sends it commands over WiFi.

The guiding rule: **be honest.** The dog only claims it "found the couch" or "got home" if a
sensor or a photo actually proves it.

## What the dog can do today
- **Recognize things it sees.** It can spot a bookshelf, a chair, the laptop, etc. through its
  camera, and tell *which direction* they're in (left/center/right) and *roughly how close*.
- **Turn by the right amount.** It has no compass, so instead of guessing angles it **watches
  how the scene slides across its camera** to measure how far it actually turned — much more
  reliable than its built-in motion sensor (which badly under-counts turns).
- **Walk to a target.** Pick a landmark (say, the bookshelf), and it steers itself toward it,
  nudging left/right to stay aimed, and stops a safe distance away.
- **Notice when it's stuck or has fallen.** If it's told to move but the view doesn't change,
  it realizes it's wedged against something, stops, and tries another direction. If it tips
  over, it detects that and refuses to keep walking (so it doesn't thrash on its side).
- **Find "home."** From anywhere, it can spin around, look for the bookshelf, and turn to face
  it — re-finding its starting spot on its own.
- **Remember the room.** It can build and save a rough map of the room from its camera and
  recognize where it is when it comes back later, instead of starting from scratch each time.
- **Record everything.** Every run is saved as a video from the dog's-eye view plus a log of
  what it was sensing, and posted online so we can review what happened.

## The plan — what each step means, how it actually works, and where it's hard
The whole approach copies how a real dog gets around with no compass and no map handed to it:
**navigate by what you can see right now, and remember places by the landmarks around them.**
Below, for each stage: *how it works*, *the real challenge*, and *how we get past it* — and an
honest note on what's already working vs. not.

### 1. Reliable "look → step → look" loop  *(building now)*
- **How it works:** about once a second the dog grabs a camera frame and runs an object detector
  that names what it sees and *where* (far-left … center … far-right). It picks its target and
  either nudges to aim — turning a small amount, *measured by how far the image slides* — or takes
  a step forward, using its distance sensor to know when it's close. If a step doesn't change the
  view, it knows it's wedged: it stops, looks for an opening, and goes around.
- **The challenge:** the detector flickers on small/distant things, and turns come up short when
  the battery sags.
- **How we get there:** average the detector over 3 frames (ignore one-off blips); small, slow,
  camera-measured turns that retry if they didn't actually move it; median-filter the noisy
  distance sensor. **Milestone:** one clean leave-home-and-return on a full charge.
- **Status:** walking-to-a-target, stuck/fall detection, camera-measured turns, and find-home all
  work individually; stitching them into one reliable round-trip is the current work.

### 2. Go to any landmark it can recognize
- **Can it now?** Partly. It already recognizes ~80 everyday things (couch, bed, chair, TV,
  laptop, bookshelf, person…) and can walk to one. The missing piece is *finding* a named one
  that isn't already dead ahead.
- **How it'll work:** on "go to the couch," it first **searches** — pans its head, then turns in
  small steps, running the detector each time — until the couch comes into view, then runs the
  Step-1 loop to it.
- **The challenge:** the detector only knows generic categories, not "the *blue* armchair," and
  the target may be somewhere it can't see from where it's standing.
- **How we get there:** for look-alikes, add color/size cues or have the AI brain confirm "yes,
  that's the one" from a photo; for out-of-sight targets, use the map (Step 3).

### 3. Build a landmark map of the room  *(the key piece — least built)*
- **How it works:** while moving, the dog logs which landmarks it sees from each spot and their
  directions, building a simple **place-graph** — *from the bookshelf, the couch is ~left; from
  the couch, the doorway is ahead.* Underneath sits the visual map it already builds from its
  camera, which lets it recognize "I've been right here before."
- **Why it matters:** this graph is its **compass substitute** — it stays oriented by recognizing
  places, not by remembering angles.
- **The challenge (honest):** the camera map loses its place during turns (motion blur), and has
  no built-in real-world scale. The place-graph layer isn't built yet, and "recognize where I am"
  works only intermittently today.
- **How we get there:** turn in small paused steps so the map keeps tracking; re-anchor by
  recognizing a landmark right after each turn; measure distance in steps (~8 cm each) plus the
  distance sensor; keep the map as "places linked by landmark sightings," which tolerates the
  tracking gaps instead of needing perfect coordinates.

### 4. Round trips & patrol
- **How it works:** a route is a list of landmark waypoints; at each one the dog finds the *next*
  landmark and goes to it (Step 2), re-recognizing the place at every waypoint to cancel out
  drift, and returns by reversing the route or re-finding home.
- **The challenge:** small errors pile up over a long route. **How:** the re-recognition at each
  waypoint resets the error each time.

### 5. Find / follow Alex
- **How it works:** scan for a person, then approach in short, re-aimed steps; the AI brain
  confirms it's *you* (not a stranger) from a frame.
- **The challenge:** you move, and the remote loop is slow (~10 s). **How:** keep each step short
  and re-detect every step so it tracks you instead of charging a stale position.

### 6. Notice what changed
- **How it works:** compare today's landmark map against the remembered one and flag what moved,
  appeared, or vanished — the original "patrol and report changes" goal.

### Constraints we design around (not excuses — these shape every step)
- **Turning is power-hungry** (a spin briefly saps the battery and weakens the legs) → **fewer,
  gentler turns**, and lean on the map so it rarely has to spin in place. A software fix, so it
  works at any charge — not a hardware crutch.
- **The remote control loop is slow (~10 s round trip)** → the fast reflexes (aiming, stuck/fall,
  turn-measuring) already run *on the dog itself*, so they don't wait on WiFi.
- **Floor-level camera** → play to what it's good at (recognizing furniture/landmarks at its eye
  level) instead of chasing an overhead view it'll never have.
- **Honesty** → it only claims "arrived" / "found you" when a sensor or photo proves it.

## Watch it (videos — dog's-eye camera view)
- **Finding home** (works):
  https://github.com/alexbhandari/pidog-public/blob/main/recordings/2026-06-16_find_home_onboard.mp4
- **Leaving and returning** (worked but cut short by the battery issue):
  https://github.com/alexbhandari/pidog-public/blob/main/recordings/2026-06-16_leave_and_return_onboard.mp4
