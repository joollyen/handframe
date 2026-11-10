# handframe

A browser-based hand-tracking workspace. MediaPipe reads both hands from a live webcam feed, and gestures summon, move, resize, shelve, and discard terminal-style windows drawn over the video.

**Live demo:** https://joollyen.github.io/handframe/ (needs a webcam; a SIM mode with scripted hands runs without one)

> Demo recording: not yet recorded.

## Why I built it

I wanted to find out how far a single consumer webcam and an in-browser hand model could go as an input device: whether gestures could be made reliable enough to manage real windows, and what breaks when you try. Each version below came from a round of testing in front of the camera, noting what failed, and changing the design to fix it.

## How it works

**Tracking pipeline.** MediaPipe HandLandmarker returns 21 landmarks per hand. Each frame they are:

1. smoothed with a One-Euro filter per coordinate (heavy smoothing when the hand is slow, little lag when it is fast),
2. extrapolated one frame ahead (`Θt0 = Θt−1 + α(Θt−1 − Θt−2)`, α = 0.5) to cancel a frame of pipeline latency,
3. re-expressed in a palm frame found by PCA over the palm landmarks, so every gesture test is independent of where the hand is, how it is rotated, and how far it is from the lens.

Gestures are scored as competing hypotheses with an early-rejection threshold rather than a chain of if-statements, so one mis-tracked finger lowers a score instead of breaking the gesture. The approach is adapted from Sridhar, Oulasvirta, and Theobalt, *Interactive Markerless Articulated Hand Motion Tracking Using RGB and Depth Data* (MPI).

**Gestures**

| Gesture | Effect |
|---|---|
| Both hands in an L shape on a diagonal (director's frame) | Creates a window inside the frame. It commits after 0.4 s and stays when you drop your hands. |
| Open palm over a window | Grabs it (outline turns green) and moves it with your hand. |
| While holding, other hand pinches or opens | Shrinks or grows the held window. |
| While holding, fast swipe | Throws the window off that edge and closes it. |
| Both hands open and pointing up for 5 s | Clears the workspace. |
| Both hands on one window, brought together | Crumples it into a ball. |
| Open both hands after a crumple | Throws the ball (see "Crumple and throw" below). |
| Point / peace sign / thumb up | Select next window / spawn a window / change the window's panel. |

**Panels.** The workspace holds at most eight windows, so there are eight panels and no two windows show the same one:

| Panel | What it shows |
|---|---|
| Telemetry | Live text readout of palm angle, span, pinch distance, and hand speed |
| Vectorscope | Chroma distribution of the camera image (BT.601 Cb/Cr) |
| Waveform | Per-column luma distribution |
| RGB parade | Per-column distribution of each color channel |
| Motion | 24×14 grid of frame-to-frame luma change, with decay |
| Optical flow | Lucas-Kanade flow vectors |
| Edges | 3×3 Sobel edge map |
| Trajectory | Path history of each hand over the last 4 s |

**Window cap and shelf.** At most 3 windows float free. Creating a fourth moves the least recently touched one into a shelf in the top-right corner, drawn as a folder with the parked windows cascaded inside. The shelf holds 5. Past that, the oldest shelved window is discarded. Reaching toward the shelf fans the stack open, and grabbing a card with an open palm pulls it back out.

**Camera constraint ladder and 60 fps.** Requesting `frameRate: {ideal: 60}` alongside `width: {ideal: 1280}` lets the browser trade resolution for frame rate. On a Logitech C922 under Linux that silently produced 640×480. The camera is now opened with `exact` width and height, trying 60 fps first and falling back step by step, so a mode either opens as requested or fails visibly:

```
720p60  →  1080p30  →  720p30  →  AUTO (ideal constraints only)
```

Detection runs from `requestVideoFrameCallback`, so inference starts as soon as a frame is decoded instead of waiting for the next render tick.

**Crumple and throw (v8).** SCAN draws a floor model in front of the camera (a flat-ground model using an assumed camera height and pitch). Pointing at a spot on the floor registers a bin there. A crumpled window can then be thrown at it with simple 3D physics (gravity and drag). Distance from the lens is estimated from the palm: MediaPipe's world landmarks give the palm's real width in centimetres, and comparing that with its width in pixels gives depth (`Z = f·W/w`). Forward hand speed from monocular depth is too noisy to use directly, so the throw's direction comes from your hand and its speed is set from what reaching the bin requires, scaled by how hard you threw. A hit files the window away. A miss breaks it into its own characters, which scatter and fall.

## Run it

Use the live demo link above, or run it locally (the webcam API needs a real origin, not `file://`):

```bash
git clone https://github.com/joollyen/handframe.git
cd handframe
python3 -m http.server 8000
# open http://localhost:8000/
```

Press **CAMERA** and allow access. It starts in **SIM** mode with scripted hands, so you can see it work before granting camera access. Tested in Chrome on Linux.

The other buttons: **SKELETON** draws the hand landmarks, **SCAN** maps the floor and arms bin registration, **RES** cycles camera modes, **FEED** switches the active window to the next unused panel, **MOSAIC** shows a pixelated copy of the video behind each window, **CLEAR ALL** removes every window.

Earlier versions are in [`versions/`](versions/) and run the same way (`http://localhost:8000/versions/v3.html`).

## Security and privacy notes

- All processing happens in the browser. Video frames are not uploaded, stored, or sent anywhere. There are no `fetch`, XHR, WebSocket, or storage calls in the code.
- The only network requests load the MediaPipe library from jsDelivr (pinned to `@mediapipe/tasks-vision@0.10.14`) and the hand model from Google's model storage.
- Those two third-party loads are the trust boundary. They are version-pinned but not yet integrity-checked (no Subresource Integrity or Content Security Policy). See Next.

## Known limitations

- Depth and floor geometry are approximations. The expected error, roughly ±10–15% at arm's length, is a calculated estimate and has not been measured. The floor model assumes a level floor and a fixed camera height and pitch. This is enough for throwing paper at a basket, not for measurement.
- The camera field of view is hard-coded for a Logitech C922 (70.4° horizontal). Other cameras will produce less accurate depth.
- Tracking degrades in low light, with busy backgrounds, and when hands overlap or leave the frame.
- Hand detection requires both hands visible for the two-hand gestures. Gesture thresholds were tuned on one person's hands and one camera setup.
- Single-file app: all code is in `index.html`. There are no automated tests.

## Version history

Each version came out of a test session on the previous one.

| Version | What changed | Why |
|---|---|---|
| v1 | Two-hand L frame boots a terminal inside the rectangle; it tracks, scales, and tilts with the hands and collapses when the gesture drops. SIM mode added. | Prove the core interaction. |
| v2 | Palm-frame normalization (PCA), One-Euro filtering, one-frame extrapolation, scored gesture hypotheses. Face detection added as an experiment. Windows needed a 5 s hold to pin. | v1 compared raw landmark distances, which broke as soon as the hand rotated or moved toward the camera. |
| v3 | Windows commit after 0.4 s and persist. Closed-fist grab, swipe to throw away, other-hand resize, point/peace/thumb-up commands. Face detection removed. | In v2 the 5 s pin hold was easy to miss, so windows were never grabbable and piled up with no way to close them. |
| v4 | Open palm replaces the fist as the grab. Window cap of 3 with a 5-slot shelf. Two-hand clear gesture (0.7 s). | MediaPipe tracks a closed fist poorly because the fingers hide each other, so grabs often never registered. An open palm is the most reliable pose. |
| v5 | Clear hold raised to 5 s. Shelf redrawn as a labelled folder with a card cascade. Detection moved to `requestVideoFrameCallback`. | Raising both hands is also how you move into the frame pose, so a short clear hold fired by accident. |
| v6 | Text feeds replaced by camera-driven panels: telemetry, vectorscope, motion, edges. | Windows showed scripted text; the panels make them show something real about the camera image. |
| v7 | Eight unique panels (added waveform, RGB parade, optical flow, trajectory), never duplicated. Camera constraint ladder with `exact` constraints. | Eight panels match the eight-window maximum. The camera was silently dropping to 640×480 when asked for 60 fps. |
| v8 | Metric hand depth from palm width, floor scan, bin registration, crumple-and-throw with physics. | Extend the workspace into the space in front of the camera. |

## My role

This was built with AI-assisted development. I came up with the concept and the gesture set, tested every version on my own webcam setup, identified what failed in each round, and decided what changed in the next one.

## Next

Planned, not built:

- Subresource Integrity and a Content Security Policy for the third-party library and model loads
- A written threat model
- Gesture-based biometric authentication with replay-attack and liveness defenses
- Automated tests running in CI
- Camera field-of-view calibration instead of the hard-coded C922 value

## License

MIT. See [LICENSE](LICENSE).
