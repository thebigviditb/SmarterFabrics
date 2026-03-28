# Squat Demo — Project Requirements Document

## Overview

A real-time squat analysis tool using a chest-mounted IMU sensor (MPU6050). The system detects squat repetitions by identifying peaks and troughs in vertical acceleration, and flags lateral imbalance by detecting unwanted left/right acceleration.

---

## Hardware Setup

- **Sensor**: MPU6050, connected to Arduino via I2C
- **Mounting**: Secured to the user's chest (sternum area), oriented with a known axis alignment
- **Connection**: USB serial to a host machine running Chrome/Edge (Web Serial API)
- **Data format**: `a/g:\t{ax}\t{ay}\t{az}\t{gx}\t{gy}\t{gz}` over serial

---

## Functional Requirements

### FR-1: Real-Time Squat Rep Detection

| Item | Detail |
|------|--------|
| **Input** | Vertical-axis accelerometer data (chest-mounted sensor) |
| **Behavior** | Identify each squat rep by detecting a **trough** (bottom of squat) and the subsequent **peak** (return to standing). One complete trough-to-peak cycle = one rep. |
| **Output** | Running rep count displayed on screen, updated the instant the user returns to standing. |

#### Detection Logic

1. Apply a low-pass filter to the vertical acceleration signal to remove noise.
2. Track signal direction changes (rising vs. falling).
3. A **trough** is registered when the filtered signal drops below a configurable threshold and then begins rising.
4. A **peak** is registered when the filtered signal rises above a configurable threshold and then begins falling.
5. Enforce a minimum time gap between consecutive troughs (e.g., 400 ms) to reject double-counts from sensor jitter.
6. One rep is counted on each confirmed peak following a confirmed trough.

### FR-2: Squat Depth Estimation

| Item | Detail |
|------|--------|
| **Input** | Vertical acceleration trough magnitude per rep |
| **Behavior** | Classify each rep's depth as **shallow**, **parallel**, or **deep** based on configurable acceleration thresholds. |
| **Output** | Per-rep depth label shown in the rep log and on a live chart. |

### FR-3: Lateral Imbalance Detection

| Item | Detail |
|------|--------|
| **Input** | Lateral-axis (left/right) accelerometer data |
| **Behavior** | During each rep (trough-to-peak window), compute the peak lateral acceleration magnitude. If it exceeds a configurable threshold, flag the rep as having lateral shift. |
| **Output** | Visual warning indicator (e.g., left/right arrow highlight) shown in real time. Per-rep lateral deviation logged with magnitude and direction. |

#### Detection Logic

1. Isolate the lateral acceleration axis based on sensor mounting orientation.
2. Within each active rep window, track max absolute lateral acceleration.
3. If max |lateral accel| > threshold (e.g., 0.15 g), flag the rep.
4. Record direction (left or right) based on sign of the peak lateral value.

### FR-4: Live Stick Figure Avatar

| Item | Detail |
|------|--------|
| **Input** | Filtered vertical acceleration, current squat phase (standing / descending / at-bottom / ascending), and lateral deviation data. |
| **Behavior** | Render a 2D stick figure on a canvas that mirrors the user's squat stance in real time. The figure's pose is driven by the sensor data — not a canned animation. |
| **Output** | A continuously updating stick figure shown prominently on screen alongside the signal charts. |

#### Stick Figure Pose Model

The stick figure consists of the following joints/segments, drawn as lines and circles:

| Segment | Points |
|---------|--------|
| Head | Circle above neck |
| Torso | Neck → Hip (the chest sensor lives here — torso angle is directly measured) |
| Upper legs | Hip → Left knee, Hip → Right knee |
| Lower legs | Left knee → Left ankle, Right knee → Right ankle |
| Upper arms | Shoulder → Left elbow, Shoulder → Right elbow |
| Lower arms | Left elbow → Left wrist, Right elbow → Right wrist |

#### Pose Mapping from Sensor Data

1. **Torso angle**: Derived directly from the sensor's pitch value. Standing upright ≈ 0°, leaning forward during squat ≈ 15–45°.
2. **Knee bend**: Inferred from the squat depth estimation (FR-2). Map the filtered vertical acceleration trough magnitude to a knee angle:
   - Standing: knees at ~170° (nearly straight)
   - Shallow squat: knees at ~120°
   - Parallel squat: knees at ~90°
   - Deep squat: knees at ~60°
3. **Hip bend**: Coupled to knee bend — as knees bend deeper, hips hinge proportionally.
4. **Lateral shift**: If lateral imbalance is detected (FR-3), offset the hip joint left or right relative to the ankles, visually showing the user leaning to one side.
5. **Arms**: Default to a forward-hold position (as if holding a barbell). Not sensor-driven.

#### Interpolation

- Joint positions are interpolated smoothly (lerp) between frames to avoid jitter.
- A configurable smoothing factor controls responsiveness vs. smoothness.

#### Visual Style

- White/light-colored lines on dark background for contrast.
- Joint circles at key points (head, shoulders, hips, knees, ankles).
- Color feedback: figure turns **green** for good form, **yellow** for shallow depth, **red** when lateral imbalance is flagged.
- Faint "ideal pose" ghost outline shown behind the figure for reference (optional, togglable).

### FR-5: Live Signal Charts

| Item | Detail |
|------|--------|
| **Signal Chart** | Scrolling time-series chart showing filtered vertical acceleration with detected peaks (green markers) and troughs (red markers). |
| **Lateral Chart** | Scrolling time-series chart showing lateral acceleration with threshold bands drawn. Exceedances highlighted in red. |
| **Rep Log Table** | Scrolling table with columns: Rep #, Depth Label, Lateral Flag (L/R/None), Lateral Magnitude, Timestamp. |

### FR-6: Session Summary

| Item | Detail |
|------|--------|
| **Trigger** | User clicks "End Session" or disconnects the sensor. |
| **Output** | Summary panel showing: total reps, average depth classification, count of laterally-flagged reps, max lateral deviation recorded, session duration. |

### FR-7: Calibration

| Item | Detail |
|------|--------|
| **Trigger** | On connect or user clicks "Calibrate". |
| **Behavior** | User stands still for ~2 seconds. System samples accelerometer to establish baseline vertical (gravity direction) and zero-offset for lateral axis. MPU6050 raw values are scaled using ±2g (accel scale factor 16384) and ±250°/s (gyro scale factor 131). |
| **Output** | Status indicator showing calibration progress and completion. |

---

## Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| **NFR-1** | End-to-end latency from sensor read to screen update < 50 ms. |
| **NFR-2** | Runs entirely in-browser — no backend server required. |
| **NFR-3** | Supports Chrome and Edge (Web Serial API). |
| **NFR-4** | Single self-contained HTML file (inline JS/CSS, CDN dependencies only). |
| **NFR-5** | Configurable thresholds exposed in a settings panel (not hardcoded magic numbers). |

---

## Configurable Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `VERTICAL_LP_ALPHA` | 0.3 | Low-pass filter smoothing factor for vertical accel |
| `TROUGH_THRESHOLD` | -0.25 g | Vertical accel value that qualifies as a squat trough |
| `PEAK_THRESHOLD` | -0.05 g | Vertical accel value that qualifies as return to standing |
| `MIN_REP_INTERVAL` | 400 ms | Minimum time between consecutive troughs |
| `LATERAL_THRESHOLD` | 0.15 g | Lateral accel magnitude that triggers an imbalance flag |
| `CALIBRATION_FRAMES` | 80 | Number of frames to average during calibration |
| `DEPTH_SHALLOW` | < 0.3 g | Trough magnitude range for shallow squat |
| `DEPTH_PARALLEL` | 0.3–0.5 g | Trough magnitude range for parallel squat |
| `DEPTH_DEEP` | > 0.5 g | Trough magnitude range for deep squat |

---

## UI Layout (Single Page)

```
┌─────────────────────────────────────────────────────────┐
│  Topbar: Title, Baud, Connect, Calibrate,                │
│          Start/Stop Session                              │
├──────────────────┬──────────────────┬───────────────────┤
│                  │                  │  Rep Count (large) │
│   Stick Figure   │  Vertical Accel  │  Depth Label       │
│   Avatar         │  Chart (scroll)  │  Lateral Warning   │
│   (main view)    │                  │  Raw values        │
│                  │                  │  Orientation        │
├──────────────────┴──────────────────┤                   │
│  Lateral Accel Chart (threshold     │  Rep Log Table     │
│  bands shown, exceedances in red)   │  (scrolling)       │
└─────────────────────────────────────┴───────────────────┘
```

---

## Sensor Axis Mapping (Chest Mount)

Assumes sensor is mounted flat against the chest, USB port pointing down:

| Body Direction | Sensor Axis | Signal During Squat |
|----------------|-------------|---------------------|
| Vertical (up/down) | Z-axis | Oscillates as user descends and ascends |
| Lateral (left/right) | X-axis | Should remain near zero — nonzero = imbalance |
| Front/back | Y-axis | Minor variation (lean), not primary concern |

> **Note**: Exact axis mapping must be confirmed during calibration and may need a rotation matrix if the sensor is tilted.

---

## Out of Scope (v1)

- Knee valgus detection (would require additional sensor on lower body)
- Barbell path tracking
- Multi-user / cloud sync
- Mobile app (browser-only for now)
- Audio/haptic feedback (future enhancement)
