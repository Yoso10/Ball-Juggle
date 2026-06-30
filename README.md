# Real-Time Juggling Counter

Which one of you and your friends is a better juggler?? Test it!


Try out our interactive 3D Juggling Tracker Program: A real-time computer-vision system that automatically detects, tracks, and counts soccer-ball juggles ("V-Flip") from a live dual-camera feed. 
The system
gives immediate on-screen feedback, detects drops to the floor, and supports
two-player gameplay — all built entirely on **classical image-processing** techniques
(no deep learning at runtime, no GPU required).

> Course: *Introduction to Image Processing* — CV course final project 

> Team: Yosef Shalev · Eylon Oren · Itamar Bahat · Bar Megidish

---

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [How It Works (Pipeline)](#how-it-works-pipeline)
- [Camera Setup](#camera-setup)
- [Configuration](#configuration)
- [Calibration Workflow](#calibration-workflow)
  - [0. HSV Color Calibration](#0-hsv-color-calibration-ball)
  - [1. Floor (12-point homography)](#3-floor-calibration--12-point-homography--key-f)
  - [2. Background (MOG2)](#1-background-mog2--key-b)
  - [3. Ball Radius](#2-ball-radius--key-s)
  - [4. Header height (optional)](#5-header-height-optional--key-h)
  - [Floor Epsilon](#floor-epsilon)
- [Controls](#controls)
- [Running the System](#running-the-system)
- [Configuration Files](#configuration-files)
- [Project Structure](#project-structure)
- [Assumptions](#assumptions)
- [Notes](#notes)

---

## Overview

The goal is to count juggles automatically and reliably under realistic indoor
conditions — changing lighting, cluttered backgrounds, and frequent occlusions
(the ball disappearing behind a leg). Rather than imitating human perception, the
system reframes each challenge into something a 2D camera and basic math can
*measure*:

- A **kick** is not "seen" — it is detected as a **velocity reversal** (a V-flip)
  in the ball's vertical motion.
- The **floor** is not "recognized" — two cameras project the ball onto a shared
  world plane via **homography**, and a touch is confirmed when both projections
  agree.

Two cameras are used:

| Camera | Role | View |
|--------|------|------|
| **Main** (Front) | Primary tracking & counting | Front-facing, straight at the player |
| **Side** (Side)   | Floor agreement & optional headers | Side-angled, facing the floor area |

The main camera runs **strict** detection (Hough only). The side camera runs
**relaxed** detection (Hough with a contour fallback) but only when the side camera
already sees the ball — this suppresses false positives while keeping recall high.


Key features:

- Dual-camera ball tracking with `main` and `side` views.
- Dynamic floor calibration from a configurable grid.
- Optional `header` camera for high-trajectory validation.
- Live `pygame` dashboard with score, state, and calibration feedback.
- Runtime mode toggle for color-only detection.

---

## Architecture

The runtime is split into three layers:

- `main.py`: orchestrates camera capture, game logic, calibration, and input.
- `Src/`: performs ball detection, motion analysis, and floor homography.
- `UI/dashboard.py`: renders the interface, overlays, and handles key events.

The system uses `DualCameraManager` for synchronized frame capture and
`BallProcessor` for per-camera detection. `JugglingCounter` applies motion logic
and drop rules, while `FloorFinder` computes and applies the floor homography.

---

## How It Works (Pipeline)

Each frame from both cameras passes through the same stages before a **HIT** or
**DROP** event is produced.

### 1. Ball Detection → `(x, y, radius)` per camera
1. **Gaussian Blur** (11×11) — suppress high-frequency sensor noise.
2. **MOG2 Background Subtraction** — adaptive Gaussian Mixture Model isolates
   moving pixels; adapts to gradual lighting changes without manual recalibration.
3. **HSV Color Thresholding** — convert BGR→HSV and threshold with calibrated
   bounds; isolating Hue makes detection robust to shadows and lighting.
4. **Bitwise-AND Fusion** — keep only pixels that are *both* moving (MOG2) *and*
   the right color (HSV). Eliminates both the "static same-color object" and the
   "moving body part" failure modes.
5. **Morphology** — Opening (3×3) removes speckle noise; Closing (11×11) fills
   holes inside the ball silhouette.
6. **Hough Circle Transform** — accept only truly round shapes within the
   calibrated radius range (rejects shoes, hands, clutter). `param2` is tuned via
   config for the sensitivity/robustness trade-off.
7. **Contour Analysis** (fallback) — if Hough fails on the secondary camera, take
   the largest contour and fit a minimum enclosing circle.
8. **Kalman Filter** — a 4D `[x, y, dx, dy]` constant-velocity filter per camera
   smooths the track and bridges short occlusions (up to ~10 frames) by predicting
   from last known velocity.

### 2. Movement Analysis & Impact Detection
- **Radius Gating** — reject detections deviating >±50% from the calibrated
  baseline radius (a fast sanity check against non-ball objects).
- **Finite-Difference Velocity** — `ΔY = Y_current − Y_previous` (positive =
  falling, negative = rising).
- **Inertia State Machine** — requires 2+ consecutive consistent frames before
  locking a `FALLING`/`RISING` state (a temporal low-pass filter against jitter).
- **V-Flip (Velocity Reversal) Detection** — a HIT is registered when the ball was
  `FALLING` and then rises for 2+ frames. This is a direct physical model of a
  kick (ball falls → contacts foot → rises).
- **First-Kick Protection** — the opening flick-up uses stricter thresholds
  (3+ rising frames, ≥15 px displacement).

### 3. Floor Detection & Drop Logic (multi-layered, redundant)
- **Dual-Camera Homography** *(primary)* — each camera's pixels map to real-world
  floor coordinates via a homography matrix (RANSAC). Both cameras project the
  ball center onto the shared ground plane; if the Euclidean distance between the
  two projections is below `floor_epsilon_cm`, the ball is confirmed **on the floor → DROP**.
- **Boundary Check** — ball X outside the playing area → DROP.
- **Velocity Freeze** — near-zero velocity for ~1 second (≈30 frames) → DROP.
- **Tracking-Loss Timeout** — both cameras lose the ball for ~1 s → game ends;
  extended to ~3 s if the ball was last seen high with upward velocity (high-kick
  immunity).
- **Retroactive Kinematic Analysis** *(safety wrapper)* — examines intervals
  between recent hits; physically impossible rapid patterns (e.g. floor-bounce
  "ghost hits", intervals < 0.35 s, or shrinking intervals from energy decay) are
  identified and subtracted from the score.

---

## Camera Setup

### Camera Roles

- **Main Camera (`main_source`)** — Side A / primary tracker.
  - Captures the player, floor, and upper airspace.
  - Used for main ball trajectory and scoring.

- **Side Camera (`side_source`)** — Side B / secondary tracker.
  - Mounted at roughly 90° to the main camera.
  - Provides floor agreement and redundancy.

- **Header Camera (`header_source`)** — optional.
  - Mounted high and angled downward.
  - Used for optional header-height calibration and high-trajectory checks.

> `top_source` is kept for backward compatibility, but `main_source` is the
> current primary front-facing camera source.

### Recommended placement

- Place the main camera directly in front of the juggler.
- Place the side camera roughly perpendicular to the main view.
- Use the header camera only when you want dedicated head-height support.

---

## Configuration

The primary static configuration is in `config.py`. preferable configuration can be achived via this file.

### Camera settings

```python
"camera": {
    "main_source": 0,
    "top_source": 1,
    "side_source": 2,
    "header_source": None,
    "width": 640,
    "height": 360
},
```

- `main_source`, `side_source`, `header_source`: webcam index or video path.
- `width`, `height`: processing resolution.
- `header_source` is optional.

### Dynamic floor calibration settings

```python
"calibration": {
    "grid_width_cm": 200.0,
    "grid_length_cm": 150.0,
    "grid_width_intervals": 3,
    "grid_length_intervals": 2
},
```

- The floor grid is generated dynamically from these values.
- The number of calibration points is `(grid_width_intervals + 1) × (grid_length_intervals + 1)`.
- With the defaults, this produces a `4 × 3 = 12` point grid.

### Runtime config files

Persistent output files are saved into `Configs/`.

- `Configs/ball_config.json`: HSV profiles for `main`, `side`, and `header`.
- `Configs/floor_calibration.json`: floor homography world points and camera correspondences.
- `Configs/runtime_config.json`: runtime settings such as `floor_epsilon_cm`.

---

## Calibration Workflow

Calibration should be performed once at the start of each session (or whenever lighting / camera position changes).

> **Note:** During calibration, MOG2 motion detection is temporarily bypassed so
> detection runs on **pure HSV color** — this lets a stationary ball be detected
> while it sits still on the floor or head.

### 0. HSV Color Calibration (ball)

Open the HSV modal and adjust bounds until the ball is solid white on a black
mask. Save the profile for the current camera. Results are written to `ball_config.json` under a
separate profile per camera (`side`, `main`).

### 1. Dynamic Floor Mapping — 12-point homography — key **`F`**

This is the most critical calibration step. Press **`F`** and place the ball at
**12 marked positions** forming a 4×3 grid on the real floor. 

press:

- `SPACE` to capture the current point.
- `BACKSPACE` to undo the last point.
- `ESC` to abort.

The world coordinates can be for example:

```
X ∈ {0, 33, 67, 100} cm   (left → right)
Y ∈ {0, 50, 100} cm       (front → back)
```

For each point the ball must be detected in **both** cameras simultaneously. After
all 12 points, two homography matrices (one per camera) are computed via RANSAC and
saved to `floor_calibration.json`. From then on, a drop is confirmed when both
cameras' world-plane projections of the ball agree within `floor_epsilon_cm`.

The grid points are generated from `config.py`, not hardcoded. The defaults
produce 12 points, but the exact count depends on `grid_width_intervals` and
`grid_length_intervals`.

### 2. Background (MOG2) — key **`B`**
Step out of frame and press **`B`** to reset and rebuild the MOG2 background model
from the current scene.

### 3. Ball Radius — key **`S`**
Place the ball on the floor and press **`S`**. The system captures 15 pure-HSV
frames, takes the **median** detected radius as the baseline, and configures the
radius gate to ±50% around it. (Median, not mean — robust to outlier frames.)

### 4. Header height (optional) — key **`H`**
If a header camera is connected, hold the ball on the player's head and press **`H`** to capture a separate radius baseline for the **top** camera, enabling head-juggle ("header") detection via relative size change.


---

## Floor Epsilon

`floor_epsilon_cm` controls how closely the two camera projections must agree
before floor contact is confirmed.

- Higher epsilon makes drop detection more forgiving.
- Lower epsilon makes floor contact stricter.

Adjust live with:

- `-` / keypad `-` → decrease by `0.5 cm`
- `=` / `+` / keypad `+` → increase by `0.5 cm`

The dashboard also includes an `EPSILON` pane for slider tuning.

---

## Controls

| Key | Action |
|-----|--------|
| `T` | Start match with 3-second countdown |
| `N` | Switch player and save score |
| `R` | Reset live session |
| `D` | Toggle color-only detection |
| `V` | Toggle black/white detection mask view |
| `G` | Toggle floor grid overlay |
| `ESC` | Exit program |
| `1` | Open HSV calibration modal |
| `B` | Reset MOG2 background |
| `S` | Calibrate ball radius |
| `H` | Calibrate optional header camera |
| `F` | Start floor calibration |
| `C` | Clear evaluation log |
| `-` / `+` | Adjust floor epsilon by `0.5 cm` |

---

## Running the System

### Install dependencies

Requires **Python 3.9+** and two connected cameras (USB webcams or video files).

```bash
# Using the standard toolchain
pip install opencv-python numpy pygame

# or with the included pyproject.toml
pip install .
```

Dependencies: `opencv-python`, `numpy`, `pygame` (for the drop whistle sound).

---

### Configure the cameras

Edit `config.py` and set the sources and resolution under `CONFIG["camera"]`.

 **Configure camera sources** in `config.py` → `CONFIG["camera"]`:
   - `main_source` and `side_source` — webcam indices (e.g. `0`, `1`) or paths to
     video files (a string path enables looping playback for testing).
   - `width` / `height` — processing resolution (default 640×360).


### Launch

```bash
python main.py
```

### Calibrate

Recommended order:

1. `1` — HSV calibration
2. `B` — background capture
3. `S` — radius calibration
4. `F` — floor calibration
5. `H` — optional header calibration

Press **`T`** to start a 3-second countdown and begin a counted game.
---

## Configuration Files

| File | Purpose |
|------|---------|
| `config.py` | Static defaults for cameras, processing, detection, and logic |
| `Configs/ball_config.json` | Saved HSV bounds for camera profiles |
| `Configs/floor_calibration.json` | Floor homography calibration data |
| `Configs/runtime_config.json` | Live runtime settings |
| `evaluation/` | Benchmark frames, ground truth, and analysis scripts |

---

## Project Structure

- `main.py` — runtime orchestration and input handling.
- `Src/` — detection, logic, and floor estimation.
- `UI/dashboard.py` — pygame dashboard and UX.
- `Utils/config_utils.py` — saved config loading and dynamic floor-point generation.
- `Configs/` — persistent HSV, floor, and runtime files.
- `evaluation/` — evaluation tooling and datasets.

---

## Assumptions

- **Static cameras** — both cameras are stationary throughout a session to keep
  the spatial calibration valid.
- **Object sphericity** — the ball stays near-spherical in flight (radius gating).
- **Color contrast** — a distinct color difference exists between ball and
  background for effective HSV thresholding.
- **Roughly constant lighting** — minimal ambient-light variation so the
  pre-calibrated HSV bounds remain valid.
- **Kinematic mimicry** — human juggling follows a predictable velocity-reversal
  pattern, allowing kicks to be counted via V-flips instead of expensive collision
  modeling.
---

## Notes

- The current code supports `main`, `side`, and optional `header` camera roles.
- Floor calibration is dynamically generated from `config.py` values.
- `main_source`, `side_source`, and `header_source` accept webcam indices or video paths.


## Project Team

* **Yosef Shalev** - [LinkedIn](https://www.linkedin.com/in/yossi-shalev/)
* **Bar Megidish** - [LinkedIn](https://www.linkedin.com/in/bar-megidish-190269214/)
* **Eylon Oren** - [LinkedIn](https://www.linkedin.com/in/eylon-oren-8976a2313/)
* **Itamar Bahat** - [LinkedIn](https://www.linkedin.com/in/itamar-bahat-997291228/)



