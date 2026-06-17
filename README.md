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

## The plan
The whole approach copies how a real dog gets around with no compass and no map handed to it:
**navigate by what you can see right now, and remember places by the landmarks around them.**
Every move is "look → find a landmark → take a step → look again," so the dog never relies on
remembering which way it's facing (the thing that always goes wrong). We build that up in stages:

1. **Rock-solid "look-step-look" loop** *(in progress).* Make the basic cycle reliable: spot a
   landmark, aim at it, step, re-check — recovering on its own when it gets stuck or wedged. A
   clean leave-home-and-return trip is the immediate milestone.
2. **Go to any landmark on command.** "Go to the couch / the doorway / the bookshelf" — it
   recognizes the thing and walks to it, re-aiming the whole way. (Walking to one target already
   works; this generalizes it to anything it can recognize.)
3. **Build a landmark map of the room.** The dog remembers what's near what ("the laptop is across
   from the bookshelf"), so it can figure out roughly where it is and plan a route between places.
   **This map is its replacement for a sense of direction** — visual memory instead of a compass.
4. **Round trips & patrol.** Leave home, visit several spots in order, and reliably come back —
   using that map to stay oriented over a long route.
5. **Find a person / follow.** Recognize Alex and walk over to him; later, follow.
6. **Notice what changed.** Compare the room to its remembered map and report what moved.

### How the plan handles the hard parts (not excuses — design choices)
- **No compass:** never trust remembered direction — re-find a landmark every step, and lean on
  the room map (steps 1 & 3). This is the core of the whole design.
- **Turning drains the battery** (spinning briefly saps power and weakens the legs): **turn less
  and turn gently** — small, slow corrections instead of big spins, and use the map so it rarely
  needs to spin in place. Fixing the power problem in *software*, so it works at any charge.
- **Drift (turns slide it sideways):** harmless because it constantly re-aims at landmarks, so
  error never piles up.
- **Floor-level camera:** play to its strength — recognizing furniture and landmarks at its eye
  level — rather than fighting for an overhead view it'll never have.

## Watch it (videos — dog's-eye camera view)
- **Finding home** (works):
  https://github.com/alexbhandari/pidog/blob/master/recordings/2026-06-16_find_home_onboard.mp4
- **Leaving and returning** (worked but cut short by the battery issue):
  https://github.com/alexbhandari/pidog/blob/master/recordings/2026-06-16_leave_and_return_onboard.mp4

*(Videos live in the main project repo; the links work if that repo is public.)*
