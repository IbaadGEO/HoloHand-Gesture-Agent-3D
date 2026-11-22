# HoloHand, tested and extended with an Earth science model

Control exploded 3D models with your bare hands through a webcam: open your hand to pull a model apart, point at a part to read about it, pinch to drag it out.

This is a fork of **[HoloHand](https://github.com/tubakhxn/HoloHand-Gesture-Agent-3D) by Tuba Khan ([@tubakhxn](https://github.com/tubakhxn))**, released under the MIT licence. The idea, the renderer, the original four models and the app design are hers. This fork adds a round of measured testing, fixes to gesture recognition and part selection, a built-in test mode, and a geological model.

## Why I forked it

I'm a geoscientist, and I wanted to branch out into learning more visual and interactive tools: things that let people explore a 3D structure directly rather than read about it. I came across Tuba's gesture projects and HoloHand stood out. The exploded-view idea maps naturally onto geology, where so much is about layers you can't see from the surface.

Before adding anything of my own, I wanted to understand how well the existing gestures actually worked, so the first part of this fork is testing.

## How I tested it

### 1. First look: a recorded, logged session

I recorded a 78-second session in Chrome trying every gesture, while logging what the app detected ten times a second. That showed the problems clearly:

- The detected gesture changed **133 times**, and **64** of those lasted 0.2 s or less, so the model kept jumping between behaviours.
- **Pinch** was detected for 14.5 s, mostly when I wasn't pinching, so parts got grabbed by accident.
- **Point** registered for only 0.2 s in total, and **peace** never switched the model.
- When my second hand dropped out for a single frame, two-hand zoom fell back to one-hand rotate and the model jolted.
- Frame rate was fine (median 40 fps, lowest 10% about 30 fps), so performance wasn't the issue.

### 2. Offline checks with synthetic hands (`npm test`)

I wrote a test that feeds made-up hand poses through the gesture engine, with no camera needed. I added each problem from the recording as a test case **before** fixing it, and confirmed those cases failed on the original code (4 of them did) and passed after the fix. It now has 19 checks, including values measured from my own hand during live tests.

### 3. Guided live tests

To get comparable numbers between runs, I used an on-screen prompt that asks for one gesture at a time for 6 seconds each (open, fist, move, point, pinch, relaxed hand, peace, two hands) and scores how often the app recognised the requested gesture. The first second of each step is ignored while the hand changes pose.

### 4. Measuring the hand directly

Point and pinch kept failing, so I logged the actual finger measurements during a live run. When pointing at the screen, the index finger points toward the camera and looks short. Even MediaPipe's 3D hand model squashes the depth, so my point and my pinch measured almost the same:

| | Index extension (tip-to-wrist ÷ knuckle-to-wrist) | Thumb-to-index gap (hand sizes) |
|---|---|---|
| Pointing at the screen | 1.32 | 0.22 |
| Pinching | 1.27 | 0.23 |

No threshold can separate those two, so I changed the gesture design rather than tuning further: **point upward** and move your fingertip over a part. The app already uses the fingertip as a cursor, so you never need to aim at the screen.

### 5. Part selection under a shaky fingertip

Even with the gesture recognised, the highlighted part felt random. Selection took the part with the single nearest sample point, re-chosen every frame, and pinch searched again at a different spot (between thumb and index). I measured it on the real scene by adding an 8 px wobble to a fixed cursor (the wobble size is an assumption, not measured):

| Model | Highlight changes per second, before → after | Selection locks within 1 s, before → after |
|---|---|---|
| Jet engine | 1.46 → 0.05 | 83% → 99% |
| Radial engine | 2.33 → 0.04 | 76% → 99% |
| Sports car | 2.82 → 0.12 | 77% → 97% |
| Brain | 1.98 → 0.02 | 77% → 99% |

### 6. Built-in test mode (`?debug`)

I turned the guided test into part of the app. Open the app with `?debug` at the end of the URL, start the camera and click **Run test**. It prompts each gesture, then shows a results table with a pass or fail per step and a **Copy results** button. Besides recognition, it checks:

- how steady the highlight is while you hold over a part
- whether the part locked
- whether pinch grabbed the part you aimed at
- whether peace switched the model exactly once
- the median 3D finger measurements for each step, for tuning

The pass marks (at least 80% recognised, at most 10% false detections on a relaxed hand, at most 0.5 highlight changes per second while aiming) are my own suggested targets, not a standard. The debug code is a separate small file that only loads with `?debug`.

## Results

Guided live tests on my hand, webcam and room lighting:

| Gesture | First guided test | Final `?debug` run |
|---|---|---|
| Open palm | 80% | ✅ 85% |
| Fist | 90% | ✅ 100% |
| Move | 20% (label only, rotation worked) | ✅ 99% |
| Point (sweep across parts) | 0% | ✅ 95% |
| Point and hold on one part | 0% | ✅ 100%, locked, 0.4 highlight changes/s |
| Pinch and drag | 0% | ✅ 97%, grabbed the part I aimed at |
| Relaxed hand (should be no gesture) | 25% false point | ✅ 5% false |
| Peace | 100%, but switched model 6 times while held | ✅ 100%, exactly one switch |
| Two hands | 84% | ✅ 83% |

The final run passed all 9 steps. These numbers come from one person with one camera, so other hands and lighting may need small threshold adjustments.

## Changes to the code

| File | Change |
|---|---|
| `src/gestureEngine.js` | Measures finger shape on MediaPipe's 3D `worldLandmarks` instead of flat image positions. Each finger remembers its last state between an up and a down threshold, so it doesn't flicker. Pinch needs a tighter thumb-index gap and 4 frames in a row. Point also counts when the index clearly reaches further than the other fingers. Thresholds are named constants at the top of the file. |
| `src/handTracking.js` | Also returns the 3D landmarks. |
| `src/scene.js` | Sticky selection (`chooseSticky`): the highlighted part stays until another is clearly closer. Clear error if a model has more parts than the shader supports (24). |
| `src/main.js` | Pinch grabs the part you aimed at (pointed at in the last 2 s, otherwise the locked part). The part's name appears next to your fingertip. Peace switches the model once per sign. Two-hand zoom waits for the second hand to stay 0.25 s, and the hand already in control keeps control. Hook for `?debug`. |
| `src/debugTest.js` | New: the guided test mode. |
| `scripts/gesture-selftest.mjs` | New: offline checks, run with `npm test`. |
| `src/models/earth.js` | New: the Earth's Interior model. |
| `index.html` | Hints updated to "Point up" and "Pinch grabs the highlighted part". |

## The geological addition: Earth's Interior

The first geological model is a cutaway of the Earth. It's in a new **Earth science** group in the model bar (key `5`). Open your hand and it comes apart like an onion: the mantle and core layers split into top and bottom halves, each showing its cross-section, while the crust breaks into pieces that fly outward.

| Layer | Depth | When exploded |
|---|---|---|
| Crust | surface to about 35 km (5–10 km under oceans, 30–70 km under continents) | breaks into 8 pieces, a simple stand-in for tectonic plates |
| Upper mantle | to 410 km | halves open furthest |
| Transition zone | 410–660 km | halves open |
| Lower mantle | 660–2,890 km | halves open |
| Outer core | 2,890–5,150 km, liquid iron and nickel | halves open slightly |
| Inner core | 5,150–6,371 km, solid iron and nickel | stays at the centre |

The layers below the crust are drawn to scale using standard depths. The crust is drawn about six times thicker than real so you can see it, and its on-screen description says so. Pointing at a layer shows a short note about it; the temperatures are given as ranges because published estimates vary.

In live `?debug` runs on this model, gestures scored similarly to the other models (8 of 9 steps passing), and pinch grabbed the layer I'd aimed at. One limit: on the closed globe the crust surrounds everything, so it overlaps the inner layers and the highlight flickered more (0.8 changes per second). Opening the model first avoids that.

Next ideas: a valley cross-section (rock layers, river deposits, channel) and a seabed or plate-boundary model.

## Run it

Needs Node.js 18+ and Chrome or Edge.

```bash
npm install
npm run dev      # open the URL Vite prints and click Start camera
npm test         # offline gesture and selection checks
```

Add `?debug` to the URL for the guided test. The camera only works on `http://localhost` or `https`. `npm install` copies the MediaPipe WASM and downloads the hand model into `public/`; if that is blocked, the app fetches both from a CDN at runtime.

## Known limitations

- The thresholds were tuned on one person's hand, camera and lighting.
- On the closed Earth globe, the crust overlaps the inner layers when you point (see above).
- On narrow windows, some part labels can sit behind the side panel.

---

## Original project notes (by Tuba Khan, updated for this fork)

Your webcam is the background, a hand skeleton and a gold ring follow your hand, and each model is a glowing point cloud that pulls apart into labelled parts so you can see what is inside. Everything runs in the browser: MediaPipe Hand Landmarker + Three.js. No backend, no API keys, nothing is uploaded.

### Models

Turbofan Jet Engine (fan, guide vanes, compressors, combustor, turbines, shaft, nozzle) ·
Radial Aircraft Engine (crankcase, 9 cylinders, pistons, rods, crankshaft, exhaust ring, supercharger) ·
Sports Car (body panels, chassis, engine, drivetrain, seats, wheels, brakes) ·
Human Brain (4 lobes, cerebellum, brainstem, corpus callosum, thalamus, hippocampus, amygdala, hypothalamus) ·
Earth's Interior (added in this fork)

### Gestures

| Gesture | Effect |
|---|---|
| Open hand slowly | Explode the model. The white arc on the gold ring shows how far. |
| Close your hand / fist | Pull it back together |
| Move or twist your hand | Rotate and tilt the model |
| Point up (index finger) | Highlight the part under your fingertip; its name shows next to it. Hold still to lock it. |
| Pinch | Grab the highlighted part, drag it out, release to snap it back |
| Two hands | Move them apart or together to zoom |
| Peace sign (hold ~0.3 s) | Jump to the next model |

### Mouse and keyboard

Drag = rotate · scroll = zoom · click a part = inspect it · `Space` = explode / assemble · `←` `→` or `1`–`5` = switch model · `R` = reset view.
Sliders in the left panel control the explosion and its intensity. The top-right buttons toggle camera background, hand skeleton, labels and fullscreen.

### Tips

- Even light and a plain background help. Keep your hand 40–80 cm from the camera, palm facing it.
- Open and close your hand slowly. The peace sign needs a short hold so it does not trigger by accident.

### Files

```
src/main.js          app loop, hand overlay, gesture -> action, labels, UI
src/handTracking.js  MediaPipe wrapper (2 hands, local files then CDN, GPU then CPU)
src/gestureEngine.js landmarks -> gesture, openness, smoothed palm / fingertip / pinch, wrist roll
src/scene.js         Three.js point-cloud renderer: explode, scatter transition, picking, part dragging
src/models/          shapes.js (point sampling) + jet / radial / car / brain / earth definitions
src/debugTest.js     ?debug guided gesture test (this fork)
scripts/             gesture-selftest.mjs, run with npm test (this fork)
```

To add a model: write a `build()` returning parts (`P(name, description, color, [I(cloud, explodeDirection)])`) and register it in `src/models/index.js`.

## Credit and licence

Original project: [tubakhxn/HoloHand-Gesture-Agent-3D](https://github.com/tubakhxn/HoloHand-Gesture-Agent-3D) by Tuba Khan, MIT licence. Her copyright notice is kept in `LICENSE`. Testing, fixes and the Earth's Interior model in this fork are by Ibaad Sidik ([@IbaadGEO](https://github.com/IbaadGEO)).
