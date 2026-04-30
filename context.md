# Project Context — Multi-Modal Aerial + Ground Surveillance System

## Project Title
Tri-Modal Surveillance: Fusing Aerial Vision, Aerial mmWave Radar, and
Ground-Based WiFi CSI Sensing for Situational Awareness in Defended Zones

## Team
- Size: 5 members
- Year: BTech Final Year (Computer Science)
- Skills: Python, JavaScript; willing to learn embedded/hardware with AI assistance
- Location: India

## Timeline
- Proposal deadline: 2026-04-30 (today)
- Project duration: 12 months
- Deliverable: Full working hardware prototype + research paper draft

## Budget
- Realistic working budget: ₹18,000–22,000 INR
  (the previously stated ₹10,000 initial budget cannot cover a flyable drone +
   sensor payload; see Bill of Materials)
- Stretch (with thermal camera + Coral accelerator): ₹26,000–30,000 INR

## Hardware Owned

- Raspberry Pi 4 (2GB RAM — confirmed). This drives the architecture decision
  to stream raw video to the backend rather than running any inference on the
  drone. The 2GB budget is sufficient for camera capture + UART parsing +
  MJPEG streaming + telemetry packaging.
- Access to 3D printer (friend's, small parts only)

---

## Vision / Inspiration

**Reference 1 — Bilawal Sidhu's "spy satellite simulator":** fused public data
(satellite imagery, live ADS-B aircraft, CCTV feeds) onto a 3D city model in a
browser. Lesson: public data + military-grade UI = powerful demo effect.

**Reference 2 — Anduril EagleEye:** AR ballistic eyewear + Qualcomm edge compute
+ sensor fusion for soldier situational awareness. We replicate at student budget
using Android phones (ARCore) + Raspberry Pi 4 + COTS sensors.

**Note:** Bilawal's "WorldView" is unrelated to the NASA Worldview repo we use;
they share a name but nothing else.

---

## Architecture: Tri-Modal Surveillance System

```
                    AERIAL PLATFORM
[Drone: RPi Cam + LD2450 mmWave + GPS + RPi 4]
              │ (MJPEG video + telemetry over WiFi)
              ▼
       [Backend Server — team laptop/PC]
              ▲
              │ (WiFi CSI features over WebSocket)
[ESP32-S3 perimeter nodes × 3]
        GROUND PLATFORM
              │
              ├──→ [War Room Dashboard]  (browser, big screen, OpenLayers)
              └──→ [Soldier Mobile App]  (Android + ARCore)
```

### Why this architecture (and not the original)

1. **Inference offloaded to backend** — RPi 4 (no GPU, no NPU) cannot run
   YOLOv8-pose at usable framerates (~1–3 FPS). The drone streams video; the
   ground-station laptop runs heavy inference at 30+ FPS. This is what real
   drone systems do.
2. **Ground-based WiFi CSI nodes (RuView)** — WiFi CSI sensing is fundamentally
   incompatible with a moving aerial platform (the platform's own motion drowns
   out the human signal). Deploying ESP32-S3 nodes at fixed perimeter points
   makes RuView a real component, not just a code reference. This is also the
   strongest novelty angle: aerial + ground multi-modal fusion.
3. **LD2450 instead of LD2410** — LD2450 outputs X/Y coordinates and velocity
   for up to 3 targets via UART; LD2410 only outputs presence/energy integers.
   The fusion MLP needs real velocity features, which LD2410 does not provide.

---

## Component 1 — Drone (Aerial Sensor Platform)

Custom quadcopter carrying camera + mmWave radar + GPS, streaming live to
the ground station.

### Drone Bill of Materials

| Item | Cost (INR) | Notes |
|---|---|---|
| F450 frame | 1,000 | Standard, plenty of mounting space |
| 4× 2212 brushless motors (920KV) | 1,600 | |
| 4× 30A ESC | 1,200 | |
| Propellers (1045) | 300 | Buy spares |
| Pixhawk 2.4.8 (ArduPilot) | 2,500 | Required for autonomous waypoints |
| 3S 3000mAh LiPo + balance charger | 2,500 | ~10 min flight time |
| M8N GPS module | 1,000 | |
| Power distribution board + wiring | 500 | |
| FPV transmitter / RC receiver | 800 | RadioMaster or FlySky 6-channel |
| **Drone subtotal** | **~11,400** | |

### Sensor Payload

| Item | Cost (INR) | Notes |
|---|---|---|
| Raspberry Pi Camera v2 | 600 | 8MP, MJPEG capable |
| HLK-LD2450 24GHz mmWave radar | 800 | X/Y + velocity for ≤3 targets via UART |
| GPS (already in drone bill) | — | Reused for telemetry |
| **Payload subtotal** | **~1,400** | |

### Companion Computer

- Raspberry Pi 4 (2GB, owned)
- Powered from drone's 5V BEC line; current draw ~2A peak
- 3D-printed mount (vibration-isolated with foam) holds RPi + camera + LD2450
- Runs (lightweight pipeline only — no on-drone inference):
  - Camera capture → MJPEG stream over WiFi to backend
  - LD2450 UART parser → JSON target coordinates + velocity to backend via WebSocket
  - **pymavlink** reads Pixhawk telemetry (GPS, battery, flight mode, armed state)
    from Pixhawk TELEM2 serial port → packages as JSON → WebSocket to backend.
    This is how drone position appears on the war room map. Required.
  - That's it. All detection, tracking, and fusion happens on the backend.

**MAVProxy** runs on the RPi 4 as a serial proxy during development:

```text
Pixhawk TELEM2 → MAVProxy on RPi 4 → pymavlink companion script
                                    → Mission Planner on laptop (for calibration)
```

This lets you configure/tune the Pixhawk from a laptop while the companion
script reads telemetry simultaneously. Essential for Phase 1 setup and PID
tuning. Can be dropped in final demo if Pixhawk is fully configured.

---

## Component 2 — Ground Perimeter (WiFi CSI Nodes via RuView)

3 × ESP32-S3 deployed at fixed perimeter points (e.g. fence corners, gate,
building edge). Each node:

- Generates / receives WiFi traffic and extracts CSI from 56 subcarriers
- Runs RuView's CSI feature extraction (multi-band fusion, phase alignment)
- Streams compressed CSI features to backend over WebSocket
- Detects: human presence, motion, breach across the perimeter line

| Item | Cost (INR) | Notes |
|---|---|---|
| 3× ESP32-S3 DevKit | 1,800 | ~₹600 each |
| 3× weatherproof enclosure (3D printed) | 0 | Use friend's printer |
| Power: USB power banks or solar | 1,500 | 3× 10000 mAh banks for demo |
| **Perimeter subtotal** | **~3,300** | |

This is the deployment of the RuView codebase — not just a code reference.

---

## Component 3 — Backend Server (Team Laptop / PC)

Runs on a team member's laptop (no dedicated hardware cost). Responsibilities:

- WebSocket hub: receives drone video + drone telemetry + CSI features from perimeter nodes
- **Heavy inference (the move from the original plan):**
  - YOLOv8-pose on incoming drone MJPEG → detections + 17 keypoints
  - ByteTrack on YOLO output → persistent track IDs
  - Behavior analysis engine (rule-based, see below)
  - Radar-vision-CSI fusion MLP → threat probability per track
  - LLM query interface (Claude API + function calling)
- SQLite database: detection events, track histories, perimeter alerts
- Pushes events to dashboard + soldier app

**Why on the laptop, not on the drone:** YOLOv8-pose needs real compute. Even
with a Coral USB accelerator, on-drone pose estimation is ~10–15 FPS; on a
modest laptop CPU it's 20–30 FPS, on any GPU it's 60+ FPS. The drone's job is
to be a sensor; the ground station's job is to think.

**Tech:** FastAPI (WebSocket + REST) + Ultralytics YOLOv8 + supervision (ByteTrack
wrapper) + SQLAlchemy + SQLite.

---

## Component 4 — War Room Dashboard

Built on NASA Worldview (React 19 + Redux + OpenLayers 10.9).

**Layers shown on the map:**
- Real-time drone GPS track (vector layer)
- Detection event markers at drone's coordinates with track IDs
- Perimeter polygon + breach indicators from CSI nodes
- Free public data overlays (no cost, makes it look like Palantir):
  - OpenSky Network (~7,000 live aircraft, ADS-B)
  - CelesTrak TLE (~180 satellites in real orbital paths)

**Side panels:**
- Live drone camera feed (MJPEG)
- Detection event list with confidence + threat score
- LLM chat box (natural language → SQL query → answer + map highlight)
- Audio + visual alarm on high-confidence detections
- NVG / FLIR shader effect on camera feed (OpenLayers/Canvas — purely aesthetic)

**Decision needed (left for finalization):** 2D OpenLayers (free, fits the
NASA Worldview base) vs CesiumJS with 3D terrain (more impressive demo, free
tier sufficient). **Default recommendation:** stick with 2D OpenLayers — it's
already wired into NASA Worldview, and a 3D map is a visual nicety, not a
research contribution.

---

## Component 5 — Soldier Mobile App (Android + ARCore)

Budget replication of Anduril EagleEye on phones soldiers already own.

**Stack:** React Native (matches team's JS skills) + ARCore via native module.

**Features:**
- Live drone camera feed (same MJPEG)
- AR detection markers via ARCore when phone points toward field
- Tactical mini-map: drone + own GPS + other soldiers + perimeter nodes
- Real-time alert push (same WebSocket feed as war room)
- Compass bearing toward nearest detected threat
- Battery + drone status
- Voice alerts via Android TTS: "Alert: person detected 50 metres northeast,
  threat 0.87"

**Cost: ₹0** — uses existing phones.

---

## AI Stack — Corrected and Realistic

### Layer 1 — Detection (on backend, not drone)

- **YOLOv8n-pose** runs on backend laptop CPU/GPU at 20–30+ FPS
- Drone streams raw MJPEG to backend; no on-drone inference (RPi 4 2GB cannot
  spare the RAM, and we are skipping the Coral accelerator)
- COCO 17-keypoint pose for posture analysis (standing / crouching / down)
- **Posture caveat:** keypoints from aerial top-down view generalize poorly
  from COCO's ground-level training set. Limit posture claims to coarse
  classes; rifle-carry / hands-raised are unreliable from above and excluded.

### Layer 2 — Tracking (on backend)
- **ByteTrack** via the `supervision` library, on top of YOLO detections
- Persistent IDs (T-01, T-02...) across frames
- Internal Kalman filter for inter-frame position prediction
- Outputs: track ID, duration, displacement rate, bbox history

### Layer 3 — Behavior Analysis (on backend, rule-based)

| Behavior | Rule | Alarm |
|---|---|---|
| Loitering | Same track ID in zone > N seconds | Yellow |
| Perimeter breach | Track enters GPS-defined polygon OR CSI node fires | Red |
| Running | Track displacement / frame > threshold | Orange |
| Group formation | 3+ track IDs converge within radius | Yellow |
| Person down | Keypoints collapse to horizontal plane | Medical |

No model training; geometric rules over tracker output.

### Layer 4 — Tri-Modal Fusion MLP (THE research contribution)

**Inputs (8 features) — corrected and grounded in real sensor outputs:**

| Feature | Source | Notes |
|---|---|---|
| `radar_target_x` | LD2450 UART | Meters from drone |
| `radar_target_y` | LD2450 UART | Meters from drone |
| `radar_velocity` | LD2450 UART | m/s, signed |
| `yolo_confidence` | YOLOv8 | 0–1 |
| `yolo_bbox_area` | YOLOv8 | Apparent target size |
| `track_duration_s` | ByteTrack | Seconds since first seen |
| `track_displacement_rate` | ByteTrack | px/s |
| `csi_perimeter_active` | RuView nodes | 0/1, set if any ground node fires |

**Architecture:** MLP, 3 hidden layers (64→32→16), sigmoid output.
Output: `threat_probability ∈ [0, 1]`. Parameters: ~3,500. Inference: ~10 µs
on backend CPU.

**Training data:** ~500–800 labeled samples collected by team in 3–4 field
sessions:
- Positive: walking, running, multiple people, person crossing perimeter
- Negative: bushes in wind, animals (if available), vehicles, empty scenes
- Hard negatives: thermal noise, swaying foliage causing false radar returns

**Publishable result:** precision/recall comparison of:
1. Vision-only baseline (YOLO + threshold)
2. Radar-only baseline (LD2450 + threshold)
3. CSI-only baseline (perimeter nodes only)
4. Tri-modal fusion MLP

The 3-vs-1 baseline comparison plus an ablation study (drop each modality, see
how performance degrades) is a publishable result for IEEE Sensors or a UAV
conference workshop.

### Layer 5 — LLM Tactical Query Interface
Claude API + function calling → SQL queries on detection event DB. Operator
asks natural-language questions, dashboard answers and highlights on map.

Examples:
- "How many people detected in the last 10 minutes?"
- "Show all detections with threat score above 0.8"
- "Where has T-03 been moving?"
- "Any perimeter breaches today?"

Effort: ~2 weeks for one person. Strong demo effect.

### Layer 6 — Trajectory Prediction
Kalman filter (extending ByteTrack's predictor) extrapolates 5 s forward.
Dotted predicted path shown on war room map and soldier AR view.

### Layer 7 — Voice Alerts on Soldier App
Android TTS triggered by same WebSocket alert events. ~1 day of work.

### REMOVED from original plan (do not include in proposal)
- ❌ "CFAR / MTI / Doppler FFT on LD2410 raw output" — LD2410 does not expose
  raw radar data; it outputs pre-processed energy values only. Same is true of
  LD2450, which outputs already-tracked target coordinates. Custom radar signal
  processing is not possible with these modules.
- ❌ "Radar vital signs from aerial platform" — micro-Doppler breathing detection
  requires a stationary radar at close range and signal-to-clutter ratios that
  a drone cannot maintain in flight. Drop this stretch goal entirely; it
  weakens the proposal's credibility if a reviewer challenges it.
- ❌ "YOLOv8-pose on RPi 4 in real-time" — replaced with backend-side inference.

---

## How Open-Source Repos Are Used (Corrected)

### NASA Worldview (`worldview/`)
- React 19 + OpenLayers 10.9 + Redux base for war room dashboard
- Add new Redux slices: drone telemetry, detection events, perimeter status
- Add vector layers for tracks, detections, perimeter polygon
- Add MJPEG sidebar component
- Optional NVG/FLIR canvas shader on camera feed

### RuView (`RuView/`) — actually deployed
- ESP32-S3 firmware for perimeter CSI nodes
- v2 Rust signal processing pipeline (multi-band, phase align, coherence gate)
  runs on backend, ingests features from each node
- Kalman pose tracker as a reference for our ByteTrack integration
- WebSocket server architecture (Axum patterns) → adapt for our FastAPI hub

### PLFM_RADAR (`PLFM_RADAR/`) — reference only, no hardware reuse
- Documentation on radar theory (LFM, pulse compression, phased arrays) for
  proposal background section — shows engineering depth in writing
- Simulation scripts (`5_Simulations/`) as classroom material for understanding
- **Not** used to "process LD2450 output" — that claim was wrong in the
  original plan and has been removed.

---

## Research Novelty (Corrected — only defensible claims)

1. **Tri-modal aerial+ground sensor fusion** — combining drone-mounted vision +
   drone-mounted mmWave radar + ground-based WiFi CSI, with all three feeding a
   single fusion MLP and unified dashboard. No prior BTech project we found
   does this combination.
2. **Ablation study of modality contribution** — quantitative measurement of
   how much each modality contributes to detection precision/recall.
3. **Behavior analysis from fused multi-modal tracks** — loitering, breach,
   running, group formation as geometric rules over tracker output.
4. **LLM natural-language tactical query interface** — Claude API + function
   calling over real-time detection database.
5. **AR-based soldier tactical interface at student budget** — replicating
   Anduril EagleEye's UX with Android + ARCore + RPi 4 sensor platform.

Citation targets: IEEE Sensors, IEEE TAES, ACM Sensys (workshop).

**Removed claims (do not put in proposal):**
- ❌ "CFAR/MTI on LD2410" (technically impossible)
- ❌ "Radar vital signs from aerial platform" (not feasible at flight altitudes)
- ❌ "Edge AI on RPi 4 for real-time YOLO-pose" (not real-time on RPi 4 CPU)

---

## Phase Plan (12 months, realistic)

| Phase | Months | Focus | Owner(s) |
|---|---|---|---|
| 1 | 1–2 | Drone build (manual RC), sensor sourcing, RPi 4 setup, individual sensor validation in lab | P1, P2 |
| 2 | 2–3 | ESP32-S3 perimeter node firmware (RuView), backend WebSocket hub skeleton, ArduPilot autonomous mode setup on validated airframe | P1, P3, P5 |
| 3 | 3–5 | YOLOv8-pose + ByteTrack on backend; behavior analysis engine; SQLite event DB | P2, P3 |
| 4 | 5–7 | Field data collection (drone flights with annotated targets); fusion MLP training & evaluation | P2, P3 |
| 5 | 6–8 | War room dashboard (NASA Worldview base + drone layer + perimeter layer + LLM query) | P4 |
| 6 | 8–10 | Soldier mobile app (React Native + ARCore + voice alerts) | P5 |
| 7 | 10–11 | Full system integration, end-to-end field testing | All |
| 8 | 11–12 | Demo polish, paper draft, final report | All |

Note: phases overlap deliberately — different team members work in parallel.

---

## Team Split (5 people)

| # | Role | Primary stack |
|---|---|---|
| P1 | Drone hardware + flight controller + ArduPilot setup | Hardware, ArduPilot Mission Planner |
| P2 | Drone-side software (camera capture, LD2450 parser, MJPEG streamer, optional MobileNet pre-filter) on RPi 4 | Python, OpenCV, TFLite |
| P3 | Backend AI: YOLOv8-pose + ByteTrack + behavior engine + fusion MLP training | Python, PyTorch, FastAPI |
| P4 | War room dashboard + LLM query interface | JS, React, OpenLayers, Claude API |
| P5 | Perimeter ESP32-S3 firmware (RuView) + Soldier mobile app (React Native + ARCore) | C/Rust (ESP32), JS (React Native) |

P5 has two responsibilities but they are sequenced (perimeter firmware in
months 2–3, mobile app in months 8–10).

---

## Bill of Materials (Final)

| Category | Item | Cost (INR) |
|---|---|---|
| Drone | Frame, motors, ESCs, props, FC, battery, GPS, PDB, RX | 11,400 |
| Sensors | RPi Cam v2, LD2450 | 1,400 |
| Perimeter | 3× ESP32-S3 + power banks | 3,300 |
| RPi 4 | already owned (2GB) | 0 |
| Spares + miscellaneous (wires, headers, foam, fasteners) | | 1,500 |
| **Working baseline** | | **~17,600** |
| Stretch: SIM800L 4G module for remote demo (optional) | | 1,500 |
| **Stretch maximum** | | **~19,100** |

**Confirmed budget approach:** ₹17,600 working / ~₹19,100 with 4G stretch.
No Coral USB Accelerator, no thermal camera — confirmed by team. Drone
streams raw MJPEG to backend over WiFi; works fine for line-of-sight demos.

---

## Key Constraints

- **Sourcing:** Robu.in, Amazon.in, AliExpress. AliExpress is slow (3–6 weeks);
  order ESP32-S3, LD2450, Pixhawk in month 1 to avoid Phase 2/3 delays.
- **DGCA drone regulations (India):** F450-class drones exceed 250g; falls
  under "Micro" or "Small" category requiring DGCA registration on the
  DigitalSky portal. Plan flight tests on private property only; obtain UIN
  before any outdoor demo.
- **WiFi range:** drone-to-backend MJPEG over 2.4GHz WiFi reliably reaches
  ~50–100m line-of-sight. Beyond that, demo needs a higher-gain antenna or a
  dedicated 5.8GHz video transmitter. Stretch goal only.
- **3D printer:** small parts only — sensor mount, camera bracket, ESP32
  weatherproof enclosure. Plan prints in month 1.
- **RPi 4 RAM:** 2GB confirmed. Architecture is locked to backend-side
  inference; drone runs lightweight capture/stream/parse only.
- **Field test location:** team's own private property (confirmed). Still file
  DGCA registration in month 1 to keep the demo defensible.

---

## Risk Register (review before finalizing)

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Drone crash damages RPi/sensors | High | High | Buy spare props, balance carefully, fly low for first 5 sessions; keep ₹2,000 reserve for replacements |
| LD2450 false positives in foliage | High | Medium | Built into MLP training set as hard negatives; CFAR-equivalent already done internally by chip |
| YOLOv8 generalizes poorly to top-down aerial views | High | High | Fine-tune on VisDrone or DOTA aerial datasets; budget Phase 4 time for this |
| ESP32-S3 CSI sensing fragile in outdoor environments | Medium | Medium | Validate indoors first; use perimeter "line crossing" rather than precise pose; rely on multiple nodes voting |
| Backend WiFi latency makes real-time AR app feel laggy | Medium | Medium | Compress detection events to small JSON; render predicted-trajectory markers via Layer 6 to mask latency |
| DGCA registration delays outdoor field tests | Medium | High | File registration in month 1; plan indoor / private-property tests as fallback |
| Field data collection insufficient for fusion MLP | Medium | High | Plan 4 sessions, not 2; augment with synthetic data; include simple data augmentation in training pipeline |
| Battery flight time too short (~10 min) for long demos | Medium | Low | Buy 2 LiPos, hot-swap; plan demo as multiple short missions |
| RPi 4 2GB constrains drone software | Confirmed | Resolved | Architecture locked to raw-stream-only; no on-drone inference. RPi handles capture + UART parse + WebSocket only — fits in 2GB comfortably |
| ArduPilot setup time exceeds estimate | Medium | Medium | Phase 1 uses manual RC (proven, simple); ArduPilot autonomous waypoints introduced in Phase 2 once airframe is validated. If Pixhawk setup stalls, manual RC remains the fallback for the entire project |

---

## Decisions Made (Locked In)

1. **RPi 4 RAM:** 2GB confirmed. No upgrade. Architecture compensates by
   running zero inference on the drone.
2. **War room map:** 2D OpenLayers via NASA Worldview base. 3D (Cesium) deferred
   to a post-research polish pass — not a research deliverable.
3. **On-drone software:** raw MJPEG stream + LD2450 UART forwarding only. No
   pre-filter, no on-drone YOLO. May revisit only if WiFi bandwidth fails in
   field testing.
4. **Coral USB Accelerator:** not purchased. Budget priority and backend handles
   all inference.
5. **Thermal camera (MLX90640):** scratched. Not part of research contribution.
6. **Flight control:** manual RC for Phase 1 (build + airframe validation).
   ArduPilot autonomous waypoints introduced in Phase 2 once airframe is proven.
   Manual RC remains the fallback if autonomous setup fails.
7. **Field test location:** team's own private property. DGCA registration
   still filed in month 1 to keep the demo defensible if anyone asks.

---

## Things That Are NOT Changing

These were correct in the original plan and remain:
- War Room dashboard built on NASA Worldview React base
- Public data overlays (OpenSky + CelesTrak) for visual richness
- Soldier app on Android + ARCore + React Native
- LLM tactical query interface via Claude API
- Behavior analysis as geometric rules over tracker output
- Trajectory prediction via Kalman filter
- Voice alerts via Android TTS
- 5-person team split with parallel workstreams
