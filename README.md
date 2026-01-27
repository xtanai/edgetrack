# EdgeTrack

EdgeTrack is designed as a deterministic capture backend for professional 3D authoring and interaction systems, not as a consumer depth camera or XR peripheral.

**EdgeTrack** is an on-edge capture and preprocessing stack for synchronized **RAW10 mono multi-camera** pipelines, targeting **Raspberry Pi 5 (primary)** or **Radxa Dragon Q6A (secondary)**.
It provides **deterministic camera I/O**, **calibration-aware undistortion/normalization**, and **on-edge stereo reconstruction** to output **true metric 3D keypoints** with time-consistent sampling.
Optionally, EdgeTrack can publish **ROI-reduced sparse 3D point clouds** and **reference features** (e.g. AprilTags, wristband, fingertips) to a host over **Gigabit LAN**.

EdgeTrack is designed to pair with **TDMStrobe**, enabling stable timing, low-latency capture, and **deterministic NIR illumination** via throw/fill channels and **A/B/C/D phase sequencing**.

> **Status:** early prototype. APIs, wiring, and schemas may change.

---

## Why EdgeTrack?

* **True 3D at the edge:** Stereo reconstruction runs on-device, producing **metric 3D keypoints** instead of raw images or 2D detections.
* **Deterministic capture:** Synchronized **global-shutter sensors** with TDM strobe phases ensure repeatable timing and geometry.
* **Low latency:** Edge-side preprocessing and triangulation minimize host-side load and end-to-end delay.
* **Bandwidth-efficient:** Transmits only **3D keypoints, sparse ROI point clouds, and references** — no raw video streaming.
* **Scalable:** 2× cameras per device (stereo); combine **3–4 stereo rigs** via LAN.
* **Marker-optional:** Gloveless by default; **wristbands** improve arm stability; **fingertip markers** enable highest precision.
* **Optional NPU assist:** Use an optional NPU to improve robustness (e.g., fast left/right disambiguation, consistency checks, and early error detection).
* **Optional RGB helper camera:** A center-mounted RGB camera can support setup workflows (visual inspection, text/marker reading, calibration aids) **without being part of the 3D reconstruction path**.

---

## Comparison

> Note: This comparison focuses on architectural capabilities and integration models.
> Feature availability may vary by configuration and firmware.

| Feature / Focus                              | ZED 2i | RealSense (Stereo) | Bumblebee® X | Leap Motion | OptiTrack | Basler Stereo | Orbbec Gemini | EdgeTrack |
|----------------------------------------------|:------:|:------------------:|:------------:|:-----------:|:---------:|:-------------:|:------------:|:--------:|
| Primary use case                              | Depth sensing / XR | Depth sensing | Stereo vision | XR hand tracking | MoCap | Stereo vision | Depth sensing | Editor authoring |
| Capture FPS (typical)                         | Low–Mid |     Low–Mid       |   Low–Mid    |    High     |   Very High    |    Low–Mid    |      Mid     |  Very High*   |
| Stereo / multi-camera                         |  Yes   |        Yes         |     Yes      |     No      |    Yes    |      Yes      |     Yes      |   Yes    |
| Multi-device fusion (native)                  | Limited|      Limited       |      No      |     No      |    Yes    |      No       |      No      |   Yes    |
| Phase-offset capture                          |  No    |        No          |      No      |     No      |    No     |      No       |      No      |   Yes    |
| Deterministic event layer                     |  No    |        No          |      No      |   Limited   |    Yes    |      No       |      No      |   Yes    |
| Editor-oriented API                           |  No    |        No          |      No      |     No      |    No     |      No       |      No      |   Yes    |
| Open-source core / SDK                        |  No    |      Partial       |      No      |     No      |    No     |      No       |   Partial    |   Yes    |
| Edge-side processing (on-device)              |  Yes   |        Yes         |     Yes      |     Yes     |    No     |      No       |     Yes      |   Yes    |
| Linux-based edge device (on-board OS)         |  No    |        No          |      No      |     No      |    No     |      No       |      No      |   Yes    |
| On-device accelerator support (NPU/GPU)       |  No    |        No          |      No      |     No      |    No     |      No       |      No      |   Yes**   |
| Expandable hardware (add-ons / upgrades)      |  No    |        No          |      No      |     No      |    No     |      No       |      No      |   Yes    |
| Typical interface                             | USB | USB | USB | USB | Ethernet | USB / GigE | USB | Ethernet |
| Typical price range                           |  $$$   |        $$          |     $$$      |      $      |   $$$$    |      $$$      |     $$       |  $$    |

\* Depends on camera selection and edge platform configuration.

\** EdgeTrack accelerator support depends on the selected edge platform (e.g., optional NPU/GPU modules).

* Note: **“Phase-offset capture”** is a key advantage. It uses time-interleaved
  capture phases across multiple rigs to improve timing stability and reduce
  occlusion. After fusion in **CoreFusion**, the system can reach an **effective
  aggregate update rate** on the order of **~1000 Hz** (configuration-dependent),
  depending on the number of rigs, per-rig capture FPS, and synchronization settings.

---

## System Overview

```
[ TDMStrobe (RP2040) ] ── TRIG A/B ─► [ EdgeTrack ] ── LAN ─► [ CoreFusion (Host PC) ] ──► MotionCoder
                               │               │                         │
                       2× MIPI-CSI        3D keypoints,           Multi-stereo fusion,
                    (global-shutter)   ROI point clouds, refs     calibration, 3D key-pose
```

* **Pi 5** ingests **RAW10 mono** from **2× MIPI-CSI** cameras (e.g. OV9281), performs **undistortion, normalization, ROI extraction**, and **stereo triangulation**.
* **TDMStrobe** provides **A/B (optional C/D)** IR pulses and **2-wire sync** for shutter/LED timing, ensuring stable Z across views.
* **AprilTag reference boards**: two removable **L-brackets** with tags on multiple faces stabilize **scale, orientation, and re-localization**.

---

## Features

* **RAW10 mono ingest** (global shutter) with NEON/GPU-assisted preprocessing
* **On-edge processing**: undistortion, normalization, ROI extraction, stereo reconstruction
* **Metric 3D keypoints** (hands, wrist, fingertips) generated directly on the edge
* **Sparse ROI point clouds** with embedded **reference features** (AprilTags, wristband, fingertip aids)
* **Hard/soft sync**: EXPOSE/FLASH input from TDMStrobe; phase-aligned capture
* **LAN publisher**: zero-copy **UDP / ZeroMQ / TCP** (configurable)

---

## Bill of Materials (BOM)

**Core**

* **Raspberry Pi 5 (8 GB RAM)** — recommended; 4 GB is possible, 16 GB is optional
* **Cooling**: Pi 5 **active cooler** (heatsink + fan)
* **Storage**: microSD (≥ 64 GB) *or* NVMe (preferred for logs)

**Cameras (Stereo per Pi)**

* 2× **global‑shutter mono** modules (e.g., OV9281 1280×800 @ up to 120 fps)
* 2× **lenses** matched to FOV (target 60–90° for precision; 120° only for bring‑up)
* **850 nm band‑pass filters** (camera‑safe IR)
* **Rigid stereo mount** with baseline ~200–300 mm (context‑dependent)

**Lighting / Sync**

* **TDMStrobe** controller based on RP2040/Pico, providing A/B phases (C/D optional). Source: [TDMStrobe](https://github.com/xtanai/tdmstrobe)
* **IR emitters**: prototype 120°; production 60° (throw) + 90° (fill)
* 2‑wire sync cables from TDMStrobe to camera rig

**Power**

* **Central 24 V PSU** sized for **all rigs + lighting** (strings × current × 1.4)
* **Buck 24→5.1 V** for each Pi 5 (≥ 5 A, low ripple). Prefer a **shared 24 V bus** to minimize wall‑warts
* Optional **UPS/hold‑up** for safe shutdown

**Reference & Aids**

* **Two aluminum L‑brackets** (removable) with **AprilTags** on multiple faces
* **Wristband markers** (optional) for arm pose stability
* **Fingertip/nail markers** (optional) for peak precision

**Cabling/Hardware**

* Short MIPI cables, quality LAN (Cat6), strain relief, black matte baffles

---

## Data Flow

1. **Capture** RAW10 frames; attach timestamps and TDM phase IDs
2. **Preprocess**: undistort, normalize, extract hand ROI
3. **Stereo reconstruction** → **metric 3D keypoints**
4. **Augment** with optional **ROI point clouds** and **reference features**
5. **Publish** over LAN to **CoreFusion**
6. **CoreFusion** performs multi-rig calibration, fusion, filtering
7. **Output to MotionCoder**: clean, low-latency **3D key-pose stream**

---

## Synchronization

* **Global-shutter cameras required**
* **TDM phases** A/B per frame (C/D optional); pulses must sit **inside exposure**
* EdgeTrack timestamps all frames using EXPOSE/FLASH signals
* Host enforces **phase consistency** and rejects cross-lit frames

---

## Configuration

* **EdgeTrack (edge)**: camera IDs, intrinsics/extrinsics, baseline, FOV
* **Lighting**: phase map, throw/fill pulse widths, guard times
* **Networking**: transport (UDP / ZeroMQ / TCP), MTU, topics
* **ROI policy**: detector thresholds, crop sizes, hysteresis
* **References**: AprilTag families, board layout, wrist/fingertip options
* **CoreFusion (host)**: rig registry, inter-rig extrinsics, fusion weights, outlier rejection, smoothing, output FPS/format

---

## Quick Start

1. Mount the **L‑brackets** with AprilTags; verify coverage from both cameras
2. Wire **TDMStrobe** (A/B sync) to the stereo rig; set Throw/Fill pulses
3. Power from the **central 24 V PSU**; feed **5.1 V** to the Pi via buck
4. Start EdgeTrack; load the rig config; check undistortion/normalization
5. Verify **histogram 70–80% FS** at target exposure; adjust strobe pulses
6. Enable **ROI** and (optional) **on‑Pi keypoints**; stream to host

---

## Best Practices

* Prefer **60–90° optics** and **baselines ≥ 0.2 m** for Z precision
* Keep **exposure short**; use strobes for illumination, not shutter time
* Fix **gain** and **gamma = 1.0** (linear); disable auto-everything
* Use **NIR band-pass filters** and rigid, thermally stable mounts
* Calibrate carefully (Charuco + bundle adjustment)

---

## Roadmap

* On‑Pi NEON/GPU kernels for faster undistort/normalize
* Lightweight on‑Pi detector for ROI without full keypointing
* **CoreFusion**: multi‑rig calibration tool (AprilTag board solve, BA), per‑rig latency compensation, timebase alignment
* Configurable ROS/ZeroMQ bridges; log‑ring buffers
* Calibration helper for L‑brackets (tag layout generator)
* MotionCoder adapter: joint remapping, confidence gating, state machine hooks

---

**Note:**

For high-precision CAD/DCC workflows, multi-view NIR systems with global-shutter
cameras and synchronized strobes provide strong advantages in accuracy, timing
stability, and drift resistance. EdgeTrack is designed specifically for these
requirements and targets deterministic authoring and interaction use cases rather
than general-purpose consumer hand-tracking or inside-out XR solutions.

---

## License

**Apache‑2.0** (code, firmware, docs). Hardware files (rig plates, brackets) may use **CERN‑OHL‑S**.

---

## Safety

* **850 nm IR is invisible and hazardous**—follow **IEC 62471** guidelines.
* **Never look into emitters.** Use black matte **baffles/shields**, aim emitters away from faces, and add **interlocks** (LEDs off on loss of sync / open cover / presence detection).
* Keep **exposure short** (strobe pulses inside the camera exposure) and the **average power low**.
* Use **850 nm band-pass filters** on cameras to reduce required LED power.

**Solution Strategies:**

* **Option A – Side/Rear placement (recommended):**
  Mount stereo pairs **left/right and slightly behind** the workspace, aimed toward the monitor/work area. Add **one or two top stereo pairs** for occlusion-free coverage. This directs IR **away from eyes** while keeping the scene well lit. Future refinement: integrate cameras cleanly by recess-mounting one pair near the center of the table and another near the back edge for a slimmer, more robust design.

* **Option B – Front placement with HMD:**
  If all stereo pairs must face forward, operate with a **closed VR headset** (no see-through optics) so eyes are **occluded**. Still use baffles and interlocks to protect bystanders without headsets.

* **Option C – IR-filtering safety glasses:**
  Use **visible-light-transmitting eyewear** that strongly attenuates **near-IR (≈ 780–950 nm)** (e.g., specified optical density at **850 nm**), so users retain normal vision while IR exposure is reduced.

* **Option D – Side-shield eyewear (“horse blinkers” idea):**
  Provide **IR-blocking safety glasses with side shields** for operators/visitors when emitters face forward. Choose eyewear rated for **near-IR attenuation** and ensure a snug fit to block off-axis light.

**Note:** 

For high-precision CAD/DCC tracking, **multi-view NIR with global-shutter cameras and synchronized strobes** typically outperforms inside-out approaches (e.g., consumer hand-tracking like **Leap Motion** or **ToF** controllers) in **accuracy, drift resistance, and latency**—making it a uniquely capable solution compared with devices such as the **Meta Quest 3** and similar systems.

### Disclaimer

Prototype hardware. Use at your own risk. Ensure eye‑safety and proper thermal design in all setups.
