# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This repo contains two independent projects:

- **`RuView/`** — WiFi-based human pose estimation using Channel State Information (CSI)
- **`PLFM_RADAR/`** — AERIS-10 open-source 10.5 GHz phased array radar

---

## RuView

Turns ESP32-S3 WiFi routers into sensors: CSI signal perturbations caused by human movement are processed into 17-keypoint COCO skeleton pose estimates and vital signs (breathing, heart rate). Dual codebase: Rust v2 (`v2/`) is the primary active implementation; Python v1 (`archive/v1/`) is the legacy reference.

### Build & Test

```bash
cd RuView

# Rust — full workspace (1,031+ tests, ~2 min)
cd v2
cargo test --workspace --no-default-features

# Rust — single crate
cargo test -p wifi-densepose-signal

# Rust — single test function
cargo test -p wifi-densepose-core types::tests::test_keypoint_confidence -- --nocapture

# Rust — check without GPU
cargo check -p wifi-densepose-train --no-default-features

# Rust — lint/format
cargo fmt --all
cargo clippy --workspace --all-targets

# Python legacy
cd archive/v1
python -m pytest tests/ -x -q
python -m pytest tests/test_basic.py::test_csi_frame_parsing -v

# Deterministic proof (must print VERDICT: PASS)
python archive/v1/data/proof/verify.py

# Dashboard
cd dashboard
npm run dev          # dev server
npm run test         # vitest unit tests
npm run test:e2e     # Playwright E2E
```

### Signal Processing Pipeline

```
ESP32 → Raw CSI (56 subcarriers, I/Q)
  → Multi-band fusion (3 channels)
  → RuvSense (14 modules): phase alignment, coherence gating, field model, tomography
  → Cross-viewpoint fusion (attention + geometry)
  → Neural inference: RuVector backbone → 17-keypoint decoder + vital signs
  → PoseEstimate { keypoints[17], confidence, vital_signs }
```

### Rust Workspace (`v2/crates/`)

Leaf crates (no internal deps): `core`, `vitals`, `wifiscan`, `hardware`, `config`, `db`

| Crate | Description |
|-------|-------------|
| `wifi-densepose-core` | Core types, traits, error types, CSI frame primitives |
| `wifi-densepose-signal` | SOTA signal processing + RuvSense multistatic sensing (14 modules) |
| `wifi-densepose-nn` | Neural network inference (ONNX, PyTorch, Candle backends) |
| `wifi-densepose-train` | Training pipeline with RuVector integration |
| `wifi-densepose-ruvector` | RuVector v2.0.4 + cross-viewpoint fusion (5 modules) |
| `wifi-densepose-hardware` | ESP32 aggregator, TDM protocol, channel hopping |
| `wifi-densepose-api` | REST API (Axum) |
| `wifi-densepose-db` | Database layer (Postgres, SQLite, Redis) |
| `wifi-densepose-mat` | Mass Casualty Assessment Tool |
| `wifi-densepose-vitals` | CSI vital sign extraction (ADR-021) |
| `wifi-densepose-wifiscan` | Multi-BSSID WiFi scanning (ADR-022) |
| `wifi-densepose-wasm` | WebAssembly bindings |
| `wifi-densepose-wasm-edge` | 60+ WASM edge modules (no_std) |
| `wifi-densepose-cli` | CLI binary (`wifi-densepose`) |
| `wifi-densepose-sensing-server` | Lightweight Axum server for sensing UI |
| `nvsim` | Deterministic NV-diamond magnetometer simulator (ADR-089) |

### RuvSense Modules (`signal/src/ruvsense/`)

| Module | Purpose |
|--------|---------|
| `multiband.rs` | Multi-band CSI frame fusion, cross-channel coherence |
| `phase_align.rs` | Iterative LO phase offset estimation, circular mean |
| `multistatic.rs` | Attention-weighted fusion, geometric diversity |
| `coherence.rs` | Z-score coherence scoring, DriftProfile |
| `coherence_gate.rs` | Accept/PredictOnly/Reject/Recalibrate gate decisions |
| `pose_tracker.rs` | 17-keypoint Kalman tracker with AETHER re-ID embeddings |
| `field_model.rs` | SVD room eigenstructure, perturbation extraction |
| `tomography.rs` | RF tomography, ISTA L1 solver, voxel grid |
| `longitudinal.rs` | Welford stats, biomechanics drift detection |
| `intention.rs` | Pre-movement lead signals (200–500 ms) |
| `cross_room.rs` | Environment fingerprinting, transition graph |
| `gesture.rs` | DTW template matching gesture classifier |
| `adversarial.rs` | Physically impossible signal detection, multi-link consistency |

### Cross-Viewpoint Fusion (`ruvector/src/viewpoint/`)

| Module | Purpose |
|--------|---------|
| `attention.rs` | CrossViewpointAttention, GeometricBias, softmax with G_bias |
| `geometry.rs` | GeometricDiversityIndex, Cramer-Rao bounds, Fisher Information |
| `coherence.rs` | Phase phasor coherence, hysteresis gate |
| `fusion.rs` | MultistaticArray aggregate root, domain events |

### RuView Key ADRs (`RuView/docs/adr/`)

43 ADRs total. Active ones:

| ADR | Title | Status |
|-----|-------|--------|
| ADR-014 | SOTA signal processing | Accepted |
| ADR-016 | RuVector training pipeline integration | Accepted — complete |
| ADR-017 | RuVector signal + MAT integration | Proposed — next target |
| ADR-024 | Contrastive CSI embedding / AETHER | Accepted |
| ADR-027 | Cross-environment domain generalization / MERIDIAN | Accepted |
| ADR-028 | ESP32 capability audit + witness verification | Accepted |
| ADR-029 | RuvSense multistatic sensing mode | Proposed |
| ADR-040 | WASM edge module collection (60 modules) | Accepted |

### Validation (ADR-028)

Run after significant changes:

```bash
cd RuView

# Must be 1,031+ passed, 0 failed
cd v2 && cargo test --workspace --no-default-features

# Must print VERDICT: PASS
python archive/v1/data/proof/verify.py

# Generate + self-verify witness bundle (must be 7/7 PASS)
bash scripts/generate-witness-bundle.sh
cd dist/witness-bundle-ADR028-*/ && bash VERIFY.sh
```

If the proof hash changes after a numpy/scipy update:
```bash
python archive/v1/data/proof/verify.py --generate-hash
python archive/v1/data/proof/verify.py
```

### ESP32 Firmware

Supported hardware (not supported: original ESP32, ESP32-C3 — single-core):

| Device | Port | Role |
|--------|------|------|
| ESP32-S3 (8MB flash) | COM7 | Primary CSI sensing node |
| ESP32-S3 SuperMini (4MB) | — | Compact CSI node |
| ESP32-C6 + MR60BHA2 | COM4 | 60 GHz mmWave vital signs |

```bash
# Provision WiFi
python firmware/esp32-csi-node/provision.py --port COM7 \
  --ssid "YourWiFi" --password "secret" --target-ip 192.168.1.20

# Monitor serial
python -m serial.tools.miniterm COM7 115200
```

Firmware build requires Windows + ESP-IDF v5.4 via Python subprocess (must strip `MSYSTEM` env vars on Git Bash). Always test with real WiFi CSI — mock mode missed the Kconfig threshold bug.

### RuView Environment Variables

| Variable | Purpose |
|----------|---------|
| `CSI_SOURCE` | `auto` \| `esp32` \| `wifi` \| `simulated` |
| `MODELS_DIR` | Path to `.rvf` model files |
| `RUST_LOG` | Logging level (`info`, `debug`, `trace`) |
| `DATABASE_URL` | PostgreSQL connection string |

### RuView Active Branch

Default: `main` | Active feature: `ruvsense-full-implementation` (PR #77)

---

## PLFM_RADAR

AERIS-10 open-source 10.5 GHz phased array radar with Pulse LFM chirps. Two variants: AERIS-10N (3 km range, 8×16 antenna) and AERIS-10E (20 km range, 32×16 antenna, 10W GaN amplifiers). Supports electronic beam steering (±45°), pulse compression, Doppler processing, MTI, and CFAR.

Hardware stack: Xilinx XC7A50T FPGA + STM32F746 MCU + ADAR1000 phase shifters + ADF4382 synthesizer.

### Build & Test

```bash
cd PLFM_RADAR

# Lint / format
ruff check .
ruff format .

# Run simulation scripts
python 5_Simulations/chirp_generation.py
python 5_Simulations/beamforming_sim.py
# Outputs → 5_Simulations/generated/ (gitignored)
```

FPGA validation is done via Vivado GUI: open `9_Firmware/9_2_FPGA/vivado_project/radar.xpr`, run behavioral simulation, inspect VCD waveforms.

### Signal Processing Pipeline (FPGA)

```
DAC → LFM chirp → TX mixer (up to 10.5 GHz) → Phased array TX
  → Echo → Phased array RX → RX mixer (down to IF) → ADC
  → FPGA: I/Q baseband → decimation/FIR → pulse compression
         → Doppler FFT → MTI (clutter) → CFAR (threshold)
  → STM32: phase shifter steering, PA control, GPS/IMU fusion
  → Python GUI: real-time target display + radar control
```

### FPGA VHDL Modules (`9_Firmware/9_2_FPGA/rtl/`)

`dac_driver.vhd`, `adc_reader.vhd`, `phase_shifter.vhd`, `pulse_compressor.vhd`, `doppler_fft.vhd`, `cfar.vhd`

### Key ICs

| IC | Role |
|----|------|
| AD9523-1 | Clock generator |
| ADF4382 | TX/RX LO synthesizer |
| ADAR1000 | 4-channel phase shifter |
| LTC5552 | Mixer |
| ADTR1107 | Front-end module |
| QPA2962 | 10W GaN PA (AERIS-10E only) |

### Python Config (`pyproject.toml`)

```toml
[tool.ruff]
target-version = "py312"
line-length = 100
```

### License

- Hardware: CERN-OHL-P (open hardware)
- Software: MIT
