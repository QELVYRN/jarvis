# Document 01: Host Infrastructure & Discovery Baseline (DOC-01-INFRA)

**Document Reference:** `JARVIS-SPEC-DOC-01`  
**Classification:** Technical Specification & System Baseline  
**Target Environment:** CachyOS Linux x86_64 (Kernel `7.3.4-1-cachyos`)  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Hardware Context

Project JARVIS is designed as an autonomous, low-latency, personal agentic operating system assistant. Unlike conventional conversational wrappers or containerized chatbots, JARVIS executes directly in the user session on bare-metal CachyOS Linux. It bridges the multimodal perceptual capabilities of the Google Gemini AI family (specifically `gemini-3.8-flash` and `gemini-3.8-live`) with the low-level system APIs of the Linux kernel, Wayland display server, and PipeWire multimedia routing graph.

To ensure deterministic performance, sub-500ms voice round-trip latency, and zero system lockups, all architectural decisions are strictly anchored to the empirical hardware discovery performed on the target host.

---

## 2. Compute & Processor Subsystem

### 2.1 Hardware Profile
- **Processor:** Intel® Core™ i3-1215U (12th Generation Alder Lake-UP3 architecture, 10nm Enhanced SuperFin process).
- **Core / Thread Topology:** 6 physical cores / 8 logical threads.
  - **2 Performance Cores (P-cores):** Golden Cove microarchitecture, 2 cores / 4 threads (logical threads 0, 1, 2, 3), base frequency 1.20 GHz, maximum Turbo Boost frequency 4.40 GHz, 1.25 MiB L2 cache per core.
  - **4 Efficient Cores (E-cores):** Gracemont microarchitecture, 4 cores / 4 threads (logical threads 4, 5, 6, 7), base frequency 0.90 GHz, maximum Turbo Boost frequency 3.30 GHz, shared 2.0 MiB L2 cache cluster.
- **Cache Hierarchy:**
  - L1 Instruction Cache: 320 KiB total.
  - L1 Data Cache: 224 KiB total.
  - L2 Cache: 4.5 MiB total.
  - L3 Cache: 10 MiB shared Intel® Smart Cache.
- **Instruction Set & Neural Acceleration:**
  - x86-64-v3 / x86-64-v4 compatible instructions.
  - AVX2, AVX-VNNI (Vector Neural Network Instructions for INT8/INT16 tensor operations).
  - Intel GNA 3.0 (Gaussian & Neural Accelerator at PCI device `00:08.0`) for ultra-low-power local speech/noise processing.
  - AES-NI, FMA3, Intel VT-x hardware virtualization.

### 2.2 Process Pinning & Scheduling Strategy
Because the Intel i3-1215U features an asymmetric hybrid architecture, careless thread scheduling will cause latency spikes if real-time audio processing is scheduled on power-saving E-cores:
- **P-Core Affinity (Cores 0–3):** Dedicated to the real-time audio pipeline, client-side Voice Activity Detection (Silero VAD v5 ONNX engine), WebSocket transmission, and screen capture streaming.
- **E-Core Affinity (Cores 4–7):** Assigned to asynchronous disk I/O, SQLite WAL indexing, background context compaction, and long-running CLI subshell tasks.

```
       +-------------------------------------------------------------------------+
       |               Intel Core i3-1215U (6 Cores / 8 Threads)                 |
       +-----------------------------------+-------------------------------------+
       |   P-Cores (Threads 0, 1, 2, 3)    |     E-Cores (Threads 4, 5, 6, 7)    |
       |  - 4.40 GHz Max Turbo             |    - 3.30 GHz Max Turbo             |
       |  - Real-time Audio Ingestion      |    - SQLite WAL Indexing            |
       |  - PipeWire Stream Capture        |    - Tool Subprocess Monitoring     |
       |  - Silero VAD Inference           |    - Vector Embeddings Compaction   |
       |  - Gemini Live WebSocket Loop     |    - Memory Cleanup & Housekeeping  |
       +-----------------------------------+-------------------------------------+
```

---

## 3. Memory & Swap Constraints

### 3.1 Observed Host Memory Profile
Empirical discovery via `free -h` and `/proc/meminfo` reveals the following real-time profile under typical user desktop usage:
- **Addressable Physical RAM:** 7.4 GiB (8.0 GB DDR4/DDR5 SO-DIMM installed).
- **Used Memory:** ~4.8 GiB (occupied by Hyprland, Noctalia shell, browser sessions, audio daemons, and system services).
- **Available Memory:** **2.6 GiB** (strictly limiting local model execution).
- **Free Memory:** ~972 MiB.
- **Buff/Cache:** ~2.9 GiB (filesystem page cache).
- **Swap Subsystem:** 7.4 GiB active swap space (`/dev/zram0` compressed RAM swap or NVMe partition), with 7.4 MiB actively paged.

### 3.2 Non-Negotiable Memory Budget & Garbage Collection Thresholds
Given the 2.6 GiB available ceiling, running large local neural networks (e.g., Llama-3 8B, Whisper Large, or dense embedding models) would trigger violent memory pressure, thrashing the zram swap and freezing the Wayland compositor.

- **JARVIS Daemon Memory Ceiling:** **<= 400 MB RSS**.
- **Working Memory Buffer:** 150–250 MB RSS typical footprint.
- **Proactive Garbage Collection Trigger:** If JARVIS process RSS crosses **450 MB**, the runtime triggers an immediate memory trim:
  1. Purges cached screen frame buffers in RAM.
  2. Executes Python `gc.collect()`.
  3. Releases SQLite memory caches via `sqlite3_db_release_memory()`.
  4. Trims glibc malloc arena via `malloc_trim(0)`.

---

## 4. Graphics & Display Acceleration

### 4.1 GPU Hardware Profile
- **Controller:** Intel Corporation Alder Lake-UP3 GT1 [UHD Graphics] (PCI `00:02.0`, rev 0c).
- **Execution Units:** 64 EUs (sub-slice graphics compute).
- **Driver Stack:** Open-source Intel `i915` kernel driver, Mesa 25.x, Intel Iris Gallium driver.
- **Supported Acceleration APIs:**
  - Vulkan 1.3.
  - OpenCL 3.0 NEO.
  - VA-API (Video Acceleration API) & Intel oneVPL via FFmpeg n9.0.2.
  - Hardware codecs: H.264, HEVC (H.265 8-bit/10-bit), VP9 Profile 0/2, and AV1 hardware decoding.

### 4.2 Display Reality & Geometry
Empirical verification via `hyprctl monitors -j` disproved standard 1080p desktop assumptions:
- **Connector:** `eDP-1` (Internal laptop panel).
- **Panel ID:** BOE 0x0A21.
- **Native Resolution:** **1366x768 pixels** (WXGA 16:9 aspect ratio).
- **Refresh Rate:** **59.994 Hz** (not 120 Hz).
- **Scale Factor:** 1.0 (unscaled, integer 1:1 pixel mapping).
- **Pixel Format:** XRGB8888.

### 4.3 Multimodal Capture Implications
- Because native resolution is 1366x768, full-screen captures via `grim` are only **1.05 megapixels** (~135 KB JPEG at 75% quality).
- Artificial downscaling to 1024x576 produces negligible token savings (~15%) while causing severe visual blur on terminal fonts and code editor syntax.
- Therefore, full-frame capture operations must run at native **1366x768**, preserving crisp legibility for the vision model.

---

## 5. Storage Subsystem & Ephemeral Volumes

### 5.1 Storage Drive Specification
- **Storage Device:** KIOXIA NVMe SSD BG5 (`/dev/nvme0n1`), PCIe Gen4 x4 DRAM-less NVMe controller.
- **Filesystem Partitioning:**
  - Root Filesystem (`/`): 100 GB ext4/btrfs partition, 13 GB used, **87 GB available** (13% utilization).
  - Home Filesystem (`/home`): 100 GB ext4/btrfs partition, 8.7 GB used, **90 GB available** (9% utilization).
- **Ephemeral RAM Volumes:**
  - `/dev/shm`: 3.8 GB tmpfs (shared memory).
  - `/run/user/1000`: 760 MB tmpfs for runtime sockets, Wayland descriptors, and temporary IPC pipes.

### 5.2 Storage Access Strategy
- All ephemeral screen captures and temporary audio buffers are stored directly in `/dev/shm/jarvis/` to avoid unnecessary SSD write wear and eliminate disk I/O bottlenecks.
- Persistent databases (episodic conversation logs, tool audit records, bookmarks) reside under `/home/Qelvyrn/.local/share/jarvis/` formatted with SQLite Write-Ahead Logging (WAL) for atomic, non-blocking disk writes.

---

## 6. Operating System & Software Stack

### 6.1 Distribution & Kernel
- **Operating System:** CachyOS Linux x86_64 (Arch Linux optimized derivative).
- **Kernel Version:** `7.3.4-1-cachyos` (`#1 SMP PREEMPT_DYNAMIC`).
- **Optimization Flags:** Built with `x86-64-v3` architecture target, low-latency BBRv3 TCP congestion control, and the BORE (Burst-Oriented Response Enhancer) or EEVDF CPU scheduler.

### 6.2 Wayland Compositor & Session Hierarchy
- **Compositor:** Hyprland 0.57.2-1.1 (`cachyos-hypr-noctalia` distribution package).
- **Session Manager:** `uwsm` (Universal Wayland Session Manager). Hyprland is managed as a systemd user session (`hyprland-session.target`).
- **Desktop Shell:** Noctalia v1.2.6 (`/usr/bin/noctalia`) with real-time IPC message interface (`noctalia msg`).
- **Wayland Utilities Present:**
  - `grim` 1.5.0 (Wayland image grabber).
  - `slurp` 1.5.0 (interactive Wayland geometry selector).
  - `wl-clipboard` 2.3.0 (`wl-copy` / `wl-paste` Wayland clipboard manager).
  - `swash` (Noctalia visual annotation utility).
  - `kitty` 0.38+ (GPU-accelerated terminal emulator).
  - `ffmpeg` n9.0.2 with Intel VA-API acceleration.
  - `sqlite3` 3.48+ with FTS5 search.

### 6.3 Python Runtime Environment
- **Host Python:** Python `3.14.10` (GCC 16.2.1).
- **Deprecation Notice:** Python 3.14 has permanently removed obsolete built-in modules including `audioop`. Any audio signal processing must rely on modern NumPy routines or standard C structs.
- **Package Management Baseline:**
  - Neither `pip` nor `uv` is installed globally in the system environment (`which pip` and `which uv` return exit code 2).
  - System package manager (`pacman`) requires root privilege and password elevation (`sudo -n pacman` fails).
  - **Resolution:** Python includes the standard `ensurepip` module (pip 26.2.2). Project virtual environments must be created via:
    ```bash
    python -m venv /home/Qelvyrn/.gemini/antigravity/scratch/jarvis/.venv
    ```
    This completely isolates JARVIS dependencies without root permissions or external bootstrapping utilities.

---

## 7. Audio Subsystem & Active Peripherals

### 7.1 Sound Architecture & Routing Graph
- **Sound Server:** PipeWire 1.6.11.
- **Session Manager:** WirePlumber 0.5.19.
- **Emulation Layers:** `pipewire-pulse` (PulseAudio emulation socket at `/run/user/1000/pulse/native`) and `pipewire-alsa`.
- **DSP Audio Driver:** Sound Open Firmware (`sof-hda-dsp`) on Intel Alder Lake PCH-P.

### 7.2 Active Peripherals Reality
Discovery via `wpctl status` and `pactl list sinks short` revealed an essential hardware fact: the user operates a Bluetooth headset rather than internal laptop speakers.
- **Default Audio Sink (Playback):** Sony WI-C100 Wireless Stereo Headset connected via BlueZ 5 / PipeWire (`@DEFAULT_AUDIO_SINK@`). In bidirectional call mode, operates as HFP/mSBC (`s16le 1ch 16000Hz`).
- **Default Audio Source (Capture):** Sony WI-C100 Headset Microphone (`@DEFAULT_AUDIO_SOURCE@`).
- **Note on Dynamic Node IDs:** PipeWire assigns node IDs ephemerally per session (e.g. 83, 1040). Hardcoding node numbers is prohibited.
- **Hardware Fallback Audio Endpoints:**
  - Microphone Fallback: `card 0 [sofhdadsp], device 7 [DMIC16kHz]` (`hw:0,7`) and `device 6 [DMIC]`.
  - Speaker Fallback: `card 0 [sofhdadsp], device 0 [HDA Analog]` (Realtek ALC256 internal stereo speakers).

### 7.3 Architectural Impact on Audio Ingestion
- Hardcoding ALSA hardware devices (such as `hw:0,7`) would bypass the user's active Bluetooth headset, causing JARVIS to fail to capture the user's speech and blasting synthesized responses through laptop speakers.
- JARVIS must capture and play audio dynamically through PipeWire abstractions using `@DEFAULT_AUDIO_SOURCE@` and `@DEFAULT_AUDIO_SINK@` via `parec` and `paplay`.

### 7.4 Integrated Video Camera
- **Device Nodes:** `/dev/video0` and `/dev/video1`.
- **Device Identity:** `Integrated_Webcam_HD: Integrate` (USB ID `0000:00:14.0-6`).
- **Formats Supported:** Motion-JPEG (MJPG) 1280x720 @ 30 fps, YUYV 4:2:2 640x480 @ 30 fps.
- **Security Policy:** Video capture is strictly on-demand. Camera streaming daemons are prohibited.

---

## 8. Virtual Input & Privilege Boundaries

### 8.1 `/dev/uinput` Direct User Access
Wayland does not permit untrusted clients to synthesize global keyboard or mouse inputs. While conventional documentation recommends installing `ydotool` and running a root background daemon (`ydotoold`), inspection of the host system revealed:
- `getfacl /dev/uinput` output:
  ```
  # file: dev/uinput
  # owner: root
  # group: input
  user::rw-
  user:Qelvyrn:rw-
  group::rw-
  mask::rw-
  other::---
  ```
- Systemd-logind automatically assigns POSIX ACL `user:Qelvyrn:rw-` to `/dev/uinput` for the active local seat.
- Executing `python -c 'import os; print(os.open("/dev/uinput", os.O_WRONLY))'` succeeds with file descriptor `10`.

### 8.2 Architectural Impact on Input Automation
- JARVIS does not require `sudo`, does not require `pacman -S ydotool`, and does not require background root daemons.
- Synthetic keystrokes, shortcuts, and mouse displacement can be injected directly from Python in user space by issuing Linux `uinput` ioctls via `evdev`.

---

## 9. Baseline Discovery Summary Table

| Subsystem | Empirical Host Reality | Architectural Constraint / Design Decision |
|:---|:---|:---|
| **CPU** | Intel Core i3-1215U (2P+4E, 8 threads) | Pin audio/VAD to P-cores (0–3); background tasks to E-cores (4–7). |
| **RAM** | 7.4 GiB physical; only 2.6 GiB available | Hard <= 400 MB RSS ceiling; no local LLMs; proactive GC at 450 MB. |
| **GPU** | Intel UHD Graphics (64 EUs, Alder Lake) | VA-API hardware acceleration via FFmpeg for frame conversions. |
| **Display** | BOE 0x0A21 on `eDP-1`, 1366x768 @ 60Hz | Native 1366x768 capture; avoid wasteful downsampling. |
| **Audio** | PipeWire 1.6; active Sony WI-C100 Headset | Dynamic `@DEFAULT_AUDIO_SOURCE@` / `@DEFAULT_AUDIO_SINK@` routing. |
| **Compositor** | Hyprland 0.57.2 under UWSM session | Native `.socket2.sock` event bus; `uwsm app --` for app launching. |
| **Desktop Shell**| Noctalia v1.2.6 with IPC message bus | `noctalia msg` for notifications, volume, and caffeine inhibition. |
| **Virtual Input**| `/dev/uinput` has user ACL `Qelvyrn:rw-` | Zero-root synthetic input via Python user-space ioctl / `evdev`. |
| **Python** | Python 3.14.10; no global pip/uv | Virtual environment bootstrapped via `python -m venv` (ensurepip). |
