# EdgeTrack

EdgeTrack is designed as a deterministic capture backend for professional 3D authoring and interaction systems.

**EdgeTrack** is an on-edge capture and preprocessing stack for synchronized **RAW10 mono multi-camera** pipelines, targeting **Raspberry Pi 5 (primary)** or **Radxa Dragon Q6A (secondary)**.
It provides **deterministic camera I/O**, **calibration-aware undistortion/normalization**, and **on-edge stereo reconstruction** to output **metric-scale 3D keypoints derived from calibrated stereo geometry**, with time-consistent sampling (error bounds depend on calibration and mechanics). Optionally, EdgeTrack can publish **ROI-reduced sparse 3D point clouds** to a host over **Gigabit LAN**.

EdgeTrack is designed to pair with **[TDMStrobe](https://github.com/xtanai/tdmstrobe)** and **[CoreFusion](https://github.com/xtanai/corefusion)** to deliver stable synchronization, low-latency capture, and **deterministic NIR illumination**. TDMStrobe provides hardware-timed strobe control aligned to camera triggers and can coordinate illumination using an **A/B/C/D phase sequence** across multiple rigs to reduce cross-talk and keep exposure consistent. CoreFusion ingests the time-aligned outputs from multiple EdgeTrack nodes over **Gigabit Ethernet** and fuses them into a coherent representation (e.g., multi-view keypoints, tracks, or sparse geometry) with predictable timing and well-defined latency.

> **Status:** early prototype. APIs, wiring, and schemas may change.

---

## Why EdgeTrack?

I’m building **[MotionCoder](https://github.com/xtanai/motioncoder)**, a semantic gesture-control layer for precise 3D authoring. During development, I found that most current hand-tracking systems prioritize real-time demos and inside-out convenience - not **repeatable, time-stable, high-precision editor workflows**, especially for fine hand articulation.

That matters to me personally: I have experience with sign language, where finger articulation and stability are critical. This led me to build an open tracking stack on Raspberry Pi 5: **EdgeTrack** (edge-side multi-camera capture + stereo reconstruction) and **[CoreFusion](https://github.com/xtanai/corefusion)** (host-side fusion across multiple EdgeTrack units).

Along the way, it became clear this architecture generalizes beyond hand tracking. It also fits demanding VR/XR and 3D workflows such as **precise tool tracking**, **high-speed scanning**, **teleoperation robotics**, and **MoCap-driven authoring** - anywhere deterministic geometry and reliable timing matter.

I briefly considered patents, but the tradeoffs didn’t make sense: high cost, legal overhead, slower iteration, and added friction for users. Instead, I’m building this **open source** under the **Apache 2.0 license** to keep it accessible, practical, and fast to improve.

---

## Features

* **Metric 3D on the edge:** On-device stereo reconstruction outputs **metric 3D keypoints** (and optional sparse ROI point clouds) rather than raw video streams or purely 2D detections.
* **RAW-first capture:** **RAW10** preserves linear sensor data at the edge, avoiding ISP-induced artifacts that reduce calibration accuracy and stereo reconstruction stability.
* **Deterministic timing:** Synchronized **global-shutter sensors** with TDM strobe phases enable repeatable capture timing and geometry.
* **850/940 nm IR-ready:** Supports **850 nm or 940 nm** illumination (sensor- and filter-dependent) for stable, ambient-robust imaging and better user comfort.
* **Linux on the edge:** A Linux-based stack enables robust deployment and long-term maintainability (drivers, networking, tooling).
* **Ethernet-first networking:** Designed for **wired Ethernet** (PoE where applicable) to deliver predictable bandwidth, low jitter, and reliable multi-rig synchronization.
* **Bandwidth-efficient output:** Transmit **3D keypoints, sparse ROI point clouds, and references** - no raw video streaming required; optional high-quality H.265 preview is available for monitoring and setup.
* **Scales across rigs:** Each device runs a **2-camera stereo pair at the edge**; combine **2–8 stereo rigs** over Ethernet to cover larger capture volumes and **significantly reduce occlusions** via multi-view coverage.
* **Low latency pipeline:** Edge-side preprocessing and triangulation reduce host load and end-to-end latency.
* **Marker-optional:** Gloveless by default; **wristbands** can improve arm stability; **fingertip markers** enable highest precision.
* **Optional tracked peripherals:** Supports tracked tools such as **VR headset markers, 3D pens/styluses, and props**, enabling high-precision workflows beyond hand tracking.
* **Optional GPU/NPU AI assist:** Used for robustness enhancements such as disambiguation, confidence estimation, and failure detection. In multi-view setups (>2 cameras), geometric redundancy typically reduces the reliance on AI-based methods.
* **Optional RGB helper camera:** A center RGB camera can support setup (visual inspection, text/marker reading, calibration aids) **without entering the reconstruction path**.

---

## Comparison of Available Hardware on the Market

> Note: This comparison focuses on architectural capabilities and integration models.
> Feature availability may vary by configuration and firmware.

| Feature / Focus                              | ZED 2i & RealSense |   Bumblebee   |   Leap Motion    | OptiTrack | Basler Stereo |    Orbbec     |     EdgeTrack    |
|----------------------------------------------|:------------------:|:-------------:|:----------------:|:---------:|:-------------:|:-------------:|:----------------:|
| Primary use case                             | Depth sensing / XR | Stereo vision | XR hand tracking |   MoCap   | Stereo vision | Depth sensing | Editor authoring |
| Capture FPS (typical)                        |         Mid        |      Mid      |       High       | Very High |      Low      |      Mid      |    Very High*    |
| Stereo / multi-camera                        |         🟢         |       🟢     |        🟡         |    🟢     |      🟢      |       🟢      |        🟢       |
| RAW10 or RAW12                               |         🔴         |       🟢     |        🔴         |    🟡     |      🟢      |       🔴      |        🟢       |
| RAW10 ingest on the edge (CPU/GPU)           |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |        🟢       |
| Native multi-device fusion                   |         🔴         |       🔴     |        🔴         |    🟢     |      🔴      |       🔴      |        🟢       |
| Phase-offset capture (TDM Module)            |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |        🟢       |
| **Deterministic event layer**                |         🔴         |       🔴     |        🔴         |  **🟢**   |      🔴      |       🔴      |      **🟢**     |
| **Editor-oriented API**                      |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |      **🟢**     |
| Open-source core                             |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |        🟢       |
| Edge-side processing (on-device)             |         🟢         |       🟢     |        🟢         |    🔴     |      🔴      |       🟢      |        🟢       |
| Linux-based edge device (on-board OS)        |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |        🟢       |
| AI On-device accelerator support (NPU/GPU)   |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |        🟢**     |
| Expandable hardware (add-ons / upgrades)     |         🔴         |       🔴     |        🔴         |    🔴     |      🔴      |       🔴      |        🟢       |
| Depth range (typical)                        |      ~0.5–6 m      |    ~0.3–5 m  |     ~0.1–1 m      | ~0.2–20 m |  ~0.2–1.0 m  |   ~0.15–10 m  |  0.1–1.2 m****  |
| Depth resolution @ 0.2 m                     |       ~2 mm        |     ~2 mm    |      ~0.5 mm      | ~<0.2 mm  |    ~0.04 mm  |    ~1 mm      |  ~0.2 mm****    |
| Depth resolution @ 0.5 m                     |       ~5 mm        |     ~5 mm    |      ~2 mm        |  ~<0.5 mm |    ~0.5 mm   |    ~4 mm      |  ~1.5 mm****    |
| Depth resolution @ 1.2 m                     |      ~15 mm        |     ~15 mm   |         -         |   ~2 mm   |     ~2 mm    |    ~15 mm     |  ~6 mm****      |
| Typical interface                            |         USB        |       USB     |        USB       | Ethernet  |  USB / GigE  |      USB      |   Ethernet      |
| Typical price range                          |         $$$$       |      $$$$$    |        $$        |   $$$$$$  |     $$$$$    |      $$$      |     $***        |

\* Depends on camera selection and edge platform configuration. Effective update rates above 1000 Hz are achieved only via **TDM phase-offset interleaving** across multiple synchronized stereo rigs (a **virtual/effective rate**), not from a single physical camera.

\** EdgeTrack accelerator support depends on the selected edge platform (e.g., optional NPU/GPU modules).

\*** Market-comparable pricing requires mass production. During early stages, off-the-shelf edge hardware (e.g., Raspberry Pi 5) and selected self-built components may be used to reduce development cost.

\**** EdgeTrack depth range and resolution values are configuration-dependent. Practical performance varies with sensor choice, lens/FOV, baseline, calibration quality, NIR illumination power/pattern (e.g., 850 nm), exposure/gain, scene texture, and the stereo matching pipeline.

* EdgeTrack is intentionally optimized for deterministic, high-precision editor workflows in the near field. For the current reference configuration, the practical “product-ready” range is ~0.1–1.2 m.

* Using two or three synchronized stereo rigs (multi-view fusion) can significantly improve robustness and repeatability (especially under occlusion and low-texture conditions) compared to a single stereo pair, and may also improve effective accuracy depending on scene geometry and noise.

* Beyond ~2 m, with wide-angle 850 nm NIR flood illumination and typical power budgets, efficiency drops and stereo matching becomes less stable (SNR decreases, disparity shrinks). For longer ranges, alternative approaches may be more suitable depending on the application - for example higher-resolution RGB cameras combined with AI-based segmentation/tracking - however this is outside EdgeTrack’s primary focus, which prioritizes repeatable, phase-stable capture and near-field authoring accuracy.



> **Phase-offset capture** is a key advantage, enabling **highest-precision authoring** through deterministic, phase-stable timing.

> **Leap Motion** uses a dual-camera hardware setup; however, the system does not expose or process stereo data as a general-purpose stereo vision pipeline.

> **OptiTrack** achieves determinism through a **centralized camera array** and **proprietary synchronization hardware**. While OptiTrack offers a **“raw grayscale”** video mode, this is primarily a **Reference Mode** for monitoring/aiming rather than a stream intended for **3D reconstruction**; it is also **not fully synchronized** and typically runs at a **lower frame rate**. **EdgeTrack**, in contrast, targets determinism **at the edge** via **distributed TDM phase-offset capture** and prioritizes **uncompressed RAW10/RAW12 sensor streams** (instead of H.264/H.265 pipelines) to preserve pixel fidelity for **stable, reproducible stereo reconstruction**.

### Notes on depth range & “resolution” figures

The depth range and resolution values in this table are **order-of-magnitude estimates** meant for architectural comparison.  
In real systems, results depend heavily on optics (FOV, focus, IR filtering), surface properties, illumination power/pattern, exposure, and matching/denoise pipelines.

#### OptiTrack (marker-based triangulation)
OptiTrack is **not a stereo depth camera**. It reconstructs 3D positions by **triangulating reflective markers** across a calibrated camera array.
With a well-designed volume (good geometry, calibration, lens choice, controlled lighting), **sub-millimeter to millimeter accuracy** is achievable.
Because the measurement principle is different from active depth cameras (stereo/ToF/structured light), the “depth resolution” numbers are **not directly comparable**.

#### Basler Stereo (industrial stereo)
Basler’s industrial stereo solutions can achieve **very high precision in the near field** (e.g., ~0.04 mm at ~0.2 m under optimal conditions).
However, **maximum precision typically comes with trade-offs**: reduced effective FPS at full depth quality, strict calibration requirements, and controlled illumination/scene texture.  
This is why the table marks Basler as **“Low” FPS (typical)** in a conservative, market-wide comparison.

#### RealSense / ZED / Orbbec (consumer/prosumer depth cameras)
Many depth-camera vendors specify accuracy as a **percentage of distance** (a common rule-of-thumb is ~1–2% of range, depending on model and conditions).
For that reason, the mm-values shown here are presented as **realistic ranges**, not best-case lab numbers, and should be interpreted as “what you can typically expect” rather than guaranteed performance.

---

## Why RAW Stereo Capture Instead of H.265

Most commercially available stereo and depth cameras rely on **H.264/H.265 video compression** to reduce bandwidth and simplify integration. While this approach is sufficient for visualization, XR previews, and general perception tasks, it introduces **lossy compression artifacts, temporal smoothing, and non-deterministic frame behavior** that negatively affect precise 3D reconstruction.

For high-accuracy stereo vision, **uncompressed RAW sensor data** is fundamentally superior. Using **RAW10** preserves the original linear pixel intensities produced by the image sensor, without spatial or temporal loss. This enables more robust stereo matching, accurate sub-pixel disparity estimation, and consistent depth reconstruction across frames.

In addition, RAW capture allows precise control over **exposure, gain, and synchronization**, which is essential for deterministic multi-camera systems. When combined with **near-infrared (NIR) illumination**, RAW stereo becomes significantly more stable under varying lighting conditions and supports reliable operation in controlled and industrial environments.

For these reasons, the project deliberately avoids H.265-based pipelines and focuses on **RAW10 stereo capture**, prioritizing determinism, precision, and authoring-grade 3D data quality over bandwidth efficiency.

> Note: H.265 is common for AI segmentation streams, but compression artifacts can bias models. In EdgeTrack, **geometry is the baseline**; AI is optional.

---

## USB vs. Ethernet vs. WLAN

USB, Ethernet, and WLAN are all commonly used to connect cameras and tracking devices, but they differ significantly in terms of determinism, scalability, and reliability.

**USB** is widely available and easy to set up, making it well suited for single-device configurations, prototyping, and consumer peripherals. However, USB is typically host-driven and shared across multiple devices on the same controller. As a result, bandwidth contention, variable latency, and timing jitter can occur, especially in multi-camera setups or under high system load. While acceptable for many use cases, these characteristics can limit predictability in time-critical pipelines.

**Ethernet** is designed for distributed and scalable systems. Each device operates independently on the network and commuates using explicit packetization, buffering, and timestamps. This enables more predictable latency, cleaner synchronization across multiple devices, and stable performance over longer cable distances. Ethernet also supports structured topologies using switches, VLANs, and Power-over-Ethernet (PoE), making it well suited for multi-rig and multi-room setups. For professional capture and deterministic processing pipelines, Ethernet is often the preferred transport layer.

**WLAN (Wi-Fi)** provides flexibility and mobility by removing physical cables, which can be advantageous in portable or rapidly reconfigurable environments. However, wireless links are inherently subject to interference, variable airtime, and changing network conditions. These factors can introduce fluctuating latency, packet loss, and jitter, which complicate synchronization and reproducibility. While modern Wi-Fi standards offer high peak bandwidth, sustained real-time performance is harder to guarantee.

In practice, USB is convenient for simple setups, Ethernet offers the most control and scalability for deterministic systems, and WLAN trades predictability for mobility and ease of deployment.

> **Note:** For tight timing, direct NIC connections are preferred. Switches usually add small latency, but can add variability under congestion; use QoS/VLAN/PTP if deterministic timing is required.

---
## Modern Architecture

### 1. Clear Separation of Capture and Processing

#### Problem

Native **MIPI CSI** camera interfaces offer excellent bandwidth, low latency, and precise timing, but they suffer from a major limitation: **very short cable lengths**, which makes larger or distributed camera setups impractical.

**USB-based** camera solutions allow longer cables, but USB is **interrupt-driven and host-dependent**, which typically results in **higher latency, increased jitter, and less deterministic timing**—especially problematic for tightly synchronized multi-camera systems.

**Ethernet/LAN-based** cameras improve distance and deployment flexibility, but many implementations still rely on **OS-level interrupts, buffering, and packet scheduling**, which are not inherently optimized for **hard real-time capture** or frame-accurate multi-camera synchronization. In practice, achieving truly deterministic behavior often requires a **real-time–tuned OS and network stack**, along with careful system-level optimization.

In addition, purpose-built **GigE / 2.5GigE machine-vision cameras** remain relatively expensive—often **€500+ per unit**—which makes large multi-camera arrays **cost-heavy** and limits scalability from a hardware-budget perspective.

At the high end, **CoaXPress** can deliver outstanding performance with direct, low-latency data paths into CPU/GPU memory. However, it comes with **very high hardware cost**, requires dedicated **frame grabbers**, and scales poorly **in terms of system cost and integration complexity**. Beyond the capture hardware itself, processing **multiple high-resolution cameras on a single host** can place a substantial load on the CPU/GPU—often pushing systems toward high-end workstation-class hardware (e.g., Threadripper-class systems) and significant engineering effort to optimize the processing pipeline.

#### Solution

EdgeTrack separates **image capture** from **high-level processing** by moving **reconstruction and preprocessing directly to the edge**. Instead of concentrating the entire workload on a single, expensive host system, each edge device performs its **local reconstruction tasks** using native MIPI CSI—where it performs best—and exports only **processed, compact 3D data** (e.g., keypoints, tool poses, sparse geometry) over the network.

This architecture preserves the **timing fidelity and signal quality** of native CSI capture while remaining **cost-efficient, scalable, and deployment-friendly**.

---

### 2. Ethernet-Native Alternative to CoaXPress

A **CoaXPress-based** camera infrastructure is a high-end solution in terms of **bandwidth and timing precision**, but it is **cost-intensive** and requires substantial **integration effort**, including dedicated **frame grabbers** and complex host-side pipelines.

Instead, comparable practical precision **for the target outputs** can be achieved through a combination of:

- **Well-defined multi-view geometry**
- **Calibrated stereo triangulation**
- **Edge-side preprocessing**
- **Early fusion in CoreFusion**

By transmitting **stable 3D primitives** (e.g., keypoints, tool poses, sparse geometry) rather than raw video streams, the system delivers **reproducible, low-jitter 3D signals** with significantly lower bandwidth requirements and reduced host-side complexity.

For the intended application, this architecture can **approach CoaXPress-class results** for **pose/keypoint accuracy and temporal stability**, while offering:

- **Simpler integration**
- **Lower hardware and maintenance costs**
- **Much better scalability**

Additional rigs can be added via **standard LAN connections**, rather than consuming limited frame-grabber channels and centralized capture resources.

---

### 3. TDM Phase-Offset Capture for Deterministic Timing

EdgeTrack uses **phase-offset global-shutter capture via Time-Division Multiplexing (TDM)**.
Instead of exposing all cameras simultaneously, multiple stereo rigs are triggered in **time-interleaved phases**.

This design:

* **Reduces occlusion**
* Improves **temporal consistency**
* Enables more **stable, repeatable input**

—especially important in **close-range, tool-centric workflows** where precision matters more than visual realism.

EdgeTrack is **markerless by default**, but supports **optional, minimal markers** when additional robustness is required.
Examples include subtle fingertip markers or markers placed directly on a tool. A “3D pencil” assisted by two small markers can provide **pen-like precision**, enabling reliable writing gestures or even **virtual keyboard interaction**.

A small **MCU-based trigger controller** generates deterministic, phase-shifted triggers for **up to eight stereo rigs** at **120 FPS per rig**.
When fused in **CoreFusion**, this results in an **effective aggregate update rate of up to ~960 Hz**, while maintaining **low jitter and high temporal stability**, depending on configuration and synchronization.


#### Why TDM Is Not Distributed Over LAN (and Why an MCU Can Still Make Sense)

A dedicated MCU (e.g., **RP2040**) is **not strictly required**—on a **Raspberry Pi 5**, TDM trigger signals can be generated locally using **hardware timers and/or DMA-driven GPIO** with sufficient precision for **120 FPS**.

The key distinction is **where timing is generated**:

* **Ethernet/LAN is excellent for data transport** (payload streaming, timestamps, configuration/control messages).
* But it is **not ideal as a real-time trigger bus**, because packet delivery depends on **OS scheduling, buffering, interrupts/NAPI, NIC behavior, and switch latency**, which introduces **variable jitter**.

Therefore, EdgeTrack uses **LAN for payload and timestamps**, while **TDM phase triggering is generated locally on each edge device** (Pi-side or MCU-side). If multiple edge devices must share a common phase reference, synchronization is handled via a **deterministic wired sync bus** (e.g., **RS-485**) or a **shared time base** (e.g., clock sync + scheduled start times), rather than sending **per-frame triggers** over the network.

> In short: **LAN transports data; the edge generates timing.**

---

## LiDAR and ToF

LiDAR and Time-of-Flight (ToF) sensors are frequently used for 3D perception, but they introduce trade-offs that can be problematic for **high-precision, close-range authoring and interaction** workflows.

**Key limitations in this context:**

* **Effective spatial resolution at close range:** Many ToF/LiDAR modules—especially compact or consumer-class devices—provide lower effective spatial detail than multi-view stereo, which limits fine hand, finger, or tool-level tracking.
* **Unstructured data for interaction:** Dense depth/point outputs are not automatically useful for control. Interaction benefits most from **stable structure** (keypoints, edges, marker geometry, ROI constraints) rather than uniform depth everywhere.
* **Temporal noise and instability:** Depth measurements can be affected by multi-path interference, surface reflectivity, ambient IR, and sensor-internal filtering, causing depth flutter and jitter—undesirable for repeatable input.
* **Multi-view scaling complexity:** Scaling active depth sensors across multiple viewpoints can increase cost and complexity due to emitter interference management (time-multiplexing, frequency/coding separation) and synchronization constraints.
* **Limited deterministic control:** Many modules behave as closed systems with fixed timing and internal depth processing, offering limited external synchronization and reduced transparency for frame-accurate multi-sensor fusion.
* **Integration and cost overhead:** High-quality LiDAR/industrial ToF systems with low noise and good stability are often expensive and may require proprietary SDKs and constrained deployment workflows.
* **Near-field suitability:** Many LiDAR/ToF systems are optimized for mid-to-long range. For desktop-scale workspaces, **stereo vision with controlled illumination** often provides higher effective precision and better repeatability.

**Short:** LiDAR/ToF sensors typically increase BOM, calibration, and integration complexity. In contrast, a simple global-shutter mono camera is easier to manufacture and scale.

**EdgeTrack design choice:** EdgeTrack deliberately favors synchronized global-shutter stereo vision with controlled NIR illumination, enabling:

* higher spatial detail at close range
* deterministic timing via external strobe and phase control
* better scalability across multiple rigs
* lower system and integration cost
* full control over the capture and reconstruction pipeline

For deterministic 3D authoring, precise hand/tool interaction, and reproducible editor workflows, multi-view stereo with explicit timing control is often a more accurate, scalable, and transparent foundation than LiDAR/ToF-based systems.

---

## Use Cases Outside MotionCoder

### Low-cost VR tracking (without an IMU in the headset)

External, camera-based tracking can enable **affordable VR** while still delivering **high positional quality**, because pose estimation is computed **outside the headset** using a deterministic multi-view setup. This avoids IMU drift and reduces BOM cost and complexity on the headset side.

**Why this matters**

* **No IMU drift:** optical tracking provides an **absolute 6DoF pose** (position + orientation) from geometry, instead of integrating IMU motion that accumulates error over time. For precision workflows, this is often a better fit than IMU-only tracking. 👉 [Pen3D](https://github.com/xtanai/pen3d) and [HandNode](https://github.com/xtanai/handnode)
* **Cheaper headset hardware:** fewer sensors and less calibration on the headset.
* **Better repeatability:** stable timing + fixed geometry yields consistent pose over time.
* **Scales to multi-device setups:** multiple tracked objects can coexist in the same capture volume.

> **Note:** The headset may not need onboard XR cameras either—if the experience is driven by a **synthetic XR scene** (e.g., rendering from a tracked **ROI point cloud / proxy geometry**) rather than real-world passthrough video. Source: 👉 [HMDone](https://github.com/xtanai/hmdone)

### High-speed 3D scanning

Scan quality depends strongly on **camera resolution, optics, baseline, calibration, and working distance**. Using **low-cost 1 MP sensors (1280×800)** with a **stereo baseline of ~160 mm** at a **working distance of ~300 mm** in a **multi-view ring setup**, the system can reliably achieve **~0.1 mm precision**, with **~0.05 mm achievable under favorable conditions** (high-quality calibration, good surface texture, stable mechanics). Higher local precision (down to **~0.02 mm**) is possible only in constrained scenarios and at reduced speed.

The key advantage is **speed and robustness**: **120 FPS** with **4–6 stereo views at ~60° FOV** enables fast, stable reconstruction with low occlusion and high repeatability.

**Why this matters**

* **High throughput:** dense temporal sampling helps capture motion and reduces the need for slow “stop-and-scan”.
* **Multi-view robustness:** multiple angles reduce occlusion and improve coverage around edges and cavities.
* **Predictable performance:** calibrated stereo geometry produces stable, reproducible depth.
* **Cost-efficient scaling:** adding views improves robustness without requiring high-end machine-vision cameras.

### Precision tool & fixture tracking (Industrial / CNC / Robotics)

External multi-view tracking can be used to **track tools, fixtures, and end-effectors** with high positional stability. With stereo baselines and calibrated geometry, the system enables **repeatable sub-millimeter pose tracking** for tasks such as **tool alignment, fixture verification, robot teaching, or calibration**—without relying on expensive encoders or integrating vision hardware inside the machine.

**Why this matters**

* **Deterministic geometry:** stable pose from calibration rather than inference-only tracking.
* **Multi-view reliability:** fewer dropouts and better robustness in cluttered workspaces.
* **No inertial drift:** absolute pose tracking instead of integrated IMU motion.
* **LAN-scalable:** expand to multiple stations or rigs with standard network infrastructure.

### Motion capture for hands, tools, and props (non-entertainment)

The system can capture **hand, finger, and tool motion** at **high frame rates** for **analysis, training, and validation**—not cinematic animation. Compared to inertial systems, optical multi-view capture provides **absolute positioning**, **no drift**, and **better repeatability**, which is valuable for **ergonomics studies, usability testing, and skill analysis**.

**Why this matters**

* **High FPS = better dynamics:** fast motions become measurable and comparable.
* **Repeatable sessions:** consistent geometry enables before/after comparisons and long-term studies.
* **Marker-optional workflow:** markerless by default, with optional minimal markers for hard cases.
* **Multi-object capture:** hands + tools + props in one coordinated reference frame.

### Metrology-light & quality inspection (near-line / in-process)

For parts that do not require full CMM accuracy, the setup can serve as a **fast, contactless inspection tool**. It enables **shape comparison, deviation visualization, and presence checks** at **~0.1 mm precision**, suitable for **near-line quality control**, **prototype validation**, or **process monitoring**—with much higher throughput than traditional scanning methods.

**Why this matters**

* **Fast feedback loops:** detect gross deviations early without interrupting production for long.
* **Sufficient precision for many checks:** ideal for “go/no-go”, fit checks, and deformation monitoring.
* **Scalable layouts:** add viewpoints where needed (edges, cavities, reflective regions).
* **Low per-sensor cost:** multi-view coverage without premium metrology camera pricing.

---

## System Overview

```
[ TDMStrobe ] ─► [ EdgeTrack ] ─► [ CoreFusion ] ─► [ MotionCoder ]
```

---

## Bill of Materials (BOM)

**Core**

* **Raspberry Pi 5 (8 GB RAM)** — recommended; 4 GB is possible, 16 GB is optional
* **Cooling**: Pi 5 **active cooler** (heatsink + fan)
* **Storage**: microSD (≥ 64 GB) *or* NVMe (preferred for logs)

**Cameras (Stereo per Pi)**

* 2× **global‑shutter mono** modules (e.g., OV9281 1280×800 @ up to 120 fps), Source: 👉 [Sensor Guide](https://github.com/xtanai/sensor-guide)
* 2× **lenses** matched to FOV (target 60–90° for precision; 120° only for bring‑up), Source: 👉 [Vision Geometry Rules](https://github.com/xtanai/geo_rules)
* **850 nm band‑pass filters** (camera‑safe IR)
* **Rigid stereo mount** with baseline ~200–300 mm (context‑dependent)

**Lighting / Sync**

* **TDMStrobe** controller based on RP2040/Pico, providing A/B phases (C/D optional), Source: 👉 [TDMStrobe](https://github.com/xtanai/tdmstrobe)

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

## Configuration

* **EdgeTrack (edge)**: camera IDs, intrinsics/extrinsics, baseline, FOV
* **Lighting**: phase map, throw/fill pulse widths, guard times
* **Networking**: transport (UDP / ZeroMQ / TCP), MTU, topics
* **ROI policy**: detector thresholds, crop sizes, hysteresis
* **References**: AprilTag families, board layout, wrist/fingertip options
* **CoreFusion (host)**: rig registry, inter-rig extrinsics, fusion weights, outlier rejection, smoothing, output FPS/format

---

## Roadmap

Coming soon. 

---

## License

**Apache‑2.0** (code, firmware, docs). Hardware files (rig plates, brackets) may use **CERN‑OHL‑S**.

---

## Safety

### NIR Illumination (850 nm vs 940 nm) & Eye Safety

* **Never look into emitters.** Use black matte **baffles/shields**, aim emitters away from faces, and add **hardware interlocks** (LEDs off on loss of sync, open covers, or presence detection).
* Keep **exposure short** (strobe pulses strictly within camera exposure) and **average irradiance low**.
* Prefer **850 nm band-pass filters** on cameras to reduce the required LED output power.
* **850 nm and 940 nm are both IR-A** and are **not inherently eye-safe**; safety depends on irradiance, geometry, duty cycle, distance, and exposure time (IEC 62471).

### Solution Strategies

**Option A — Prefer more viewpoints over more power (recommended for 940 nm)**
- If **940 nm illumination** is preferred (reduced visible glow), the recommended approach is to **increase the number of stereo rigs (viewpoints)** to maintain SNR while keeping **irradiance low**, rather than compensating with higher-power NIR emitters.

**Option B — Side / rear placement (recommended)**
- Mount stereo pairs **left/right and slightly behind** the workspace, aimed toward the work area. Add **one or two top stereo pairs** for occlusion-free coverage. This directs NIR **away from the eyes** while maintaining uniform scene illumination.
Future refinement: recess-mount one stereo pair near the table center and another near the back edge for a slimmer, more robust setup.

**Option C — Front placement with HMD only**
- If stereo pairs must face forward, operate with a **closed VR headset** (no see-through optics) so the user’s eyes are **occluded**. Baffles and interlocks are still required to protect bystanders.

**Option D — IR-filtering safety eyewear**
- Use **visible-light-transmitting eyewear** that strongly attenuates **near-IR (≈ 780–950 nm)** (specified optical density at **850 nm / 940 nm**) so users retain normal vision while IR exposure is reduced.

**Option E — Side-shield eyewear (“horse-blinkers” concept)**
- Provide **IR-blocking safety glasses with side shields** for operators and visitors when emitters face forward. Ensure proper **near-IR attenuation ratings** and a snug fit to block off-axis radiation. and ensure a snug fit to block off-axis light.

---

## Disclaimer

Prototype hardware. Use at your own risk. Ensure eye‑safety and proper thermal design in all setups.
