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

## What's still hard (the honest part)
- **No sense of direction.** With no compass, it's easy for the dog to lose track of which way
  it's facing — so it has to keep re-finding landmarks rather than remembering "home is that way."
- **The camera is on the floor.** It mostly sees furniture legs and clutter, not a nice overview,
  which makes some things hard to recognize.
- **Turning is a battery hog.** Spinning in place pulls so much power that the battery voltage
  briefly sags, the legs lose strength, and the turn comes up short. So the trickier, turn-heavy
  moves only work well on a nearly full charge.
- **Turns also slide it sideways a little**, so its exact position drifts over a long sequence of
  moves — even though it can still re-aim at a landmark.

## What's next
1. Charge it full and have it do a clean **leave-home-and-come-back** trip, recorded.
2. Make turning gentler on the battery so it works at lower charge.
3. Teach it to **find a person** (e.g., walk over to Alex) and to **patrol** the room without
   getting stuck.
4. A tiny **$3 compass chip** would fix the "which way am I facing" problem and is the single
   biggest easy upgrade.

## Watch it (videos — dog's-eye camera view)
- **Finding home** (works):
  https://github.com/alexbhandari/pidog/blob/master/recordings/2026-06-16_find_home_onboard.mp4
- **Leaving and returning** (worked but cut short by the battery issue):
  https://github.com/alexbhandari/pidog/blob/master/recordings/2026-06-16_leave_and_return_onboard.mp4

*(Videos live in the main project repo; the links work if that repo is public.)*
