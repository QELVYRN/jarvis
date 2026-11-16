# Project JARVIS: Personal Agentic Operating System Assistant
### Phase 0: Infrastructure Discovery, Verification & Architecture Specifications

**Role:** Principal Software Architect & Lead AI Engineer  
**Host Target:** CachyOS Linux x86_64 (Kernel `7.3.4-1-cachyos`), Hyprland 0.57.2 Wayland, Intel Core i3-1215U, 8GB RAM  
**AI Backend Engine:** Gemini 3.8 Flash (`gemini-3.8-flash`) & Gemini Live API (`gemini-3.8-live` / `gemini-3.8-live-extended-thinking`)  
**Status:** **PHASE 0 DISCOVERY & ARCHITECTURE VERIFIED — PENDING HUMAN APPROVAL BEFORE IMPLEMENTATION**  

---

## 1. Executive Summary

**Project JARVIS** is an autonomous, low-latency, personal agentic operating system assistant engineered natively for CachyOS Linux and the Hyprland Wayland compositor. Rather than running in a restricted sandbox or operating as a superficial chat window, JARVIS interfaces directly with Linux kernel primitives, Wayland event buses, PipeWire multimedia graphs, and the Noctalia desktop shell.

JARVIS leverages Google's state-of-the-art **Gemini 3.8** model family:
- **`gemini-3.8-live`:** Powers a persistent, full-duplex, bidirectional voice loop delivering sub-500ms conversational turn latency, spoken fillers, and non-blocking background tool triggers.
- **`gemini-3.8-flash`:** Provides deep deliberative reasoning, multi-step workspace planning, large code refactoring, and calibrated safety evaluations.

All architectural decisions in this repository have been strictly verified against the empirical host hardware and operating system environment. In strict adherence to project directives, **zero implementation code has been written** during Phase 0; this repository serves as the definitive engineering blueprint awaiting formal user authorization.

---

## 2. High-Level Architecture Overview

JARVIS is built on a **three-tier asymmetric cognitive topology** designed to balance real-time conversational responsiveness with deep reasoning while respecting tight hardware memory constraints (2.6 GiB available RAM):

```
       +-------------------------------------------------------------------------+
       |                          USER INTERACTION MODES                         |
       |  [PTT: SUPER+G]   [VAD Audio: Bluetooth/DMIC]   [Screen: 1366x768]      |
       +--------------------+-------------------+-------------------+------------+
                            |                   |                   |
                            v                   v                   v
       +-------------------------------------------------------------------------+
       |                  LOCAL PERCEPTION & INGESTION LAYER                     |
       |  +--------------------+  +--------------------+  +--------------------+ |
       |  | PipeWire Audio In  |  | Hyprland .socket2  |  | Grim Memory Pipe   | |
       |  | - parec (16kHz PCM)|  | - Real-time events |  | - Active Win Crop  | |
       |  | - Silero VAD gate  |  | - Focus / Workspace|  | - 1366x768 JPEG   | |
       |  +---------+----------+  +---------+----------+  +---------+----------+ |
       +------------|-----------------------|-----------------------|------------+
                    |                       |                       |
                    +-----------------------+-----------------------+
                                            |
                                            v
       +-------------------------------------------------------------------------+
       |                     JARVIS CORE ORCHESTRATOR (PYTHON)                   |
       |  +-------------------------------------------------------------------+  |
       |  | Event Bus (asyncio) & Fast Reactive Loop                          |  |
       |  | - Live API WebSocket Manager (`google-genai` 3.2.0)               |  |
       |  | - Audio Ingestion Ring Buffer (16 kHz PCM)                        |  |
       |  | - Playback Pipeline (24 kHz PCM -> `paplay`)                      |  |
       |  | - Real-time Barge-In & State Machine Arbitrator                   |  |
       |  +----------------------------------+--------------------------------+  |
       +-------------------------------------|-----------------------------------+
                                             |
                   +-------------------------+-------------------------+
                   |                                                   |
                   v                                                   v
       +------------------------+                          +------------------------+
       |  REACTIVE LIVE ENGINE  |                          |  DELIBERATIVE AGENT    |
       |  `gemini-3.8-live`     |                          |  `gemini-3.8-flash`    |
       |  - Bidirectional Audio |                          |  - Multi-step Planning |
       |  - Low-latency Voice   |                          |  - Complex Code Gen    |
       |  - Async Non-blocking  |                          |  - Deep Reasoning      |
       |    Tool Declarations   |                          |  - HITL Safety Checks  |
       +-----------+------------+                          +-----------+------------+
                   |                                                   |
                   +-------------------------+-------------------------+
                                             |
                                             v
       +-------------------------------------------------------------------------+
       |                    ACTION & TOOL EXECUTION BROKER                       |
       |  +------------------+  +------------------+  +------------------------+ |
       |  | Hyprland IPC     |  | Noctalia IPC     |  | Subprocess Runner      | |
       |  | - hyprctl        |  | - noctalia msg   |  | - Async Sandboxed CLI  | |
       |  | - Window Focus   |  | - OSD / Caffeine |  | - Timeout & Buffer Cap | |
       |  +------------------+  +------------------+  +------------------------+ |
       |  +------------------+  +------------------+  +------------------------+ |
       |  | Direct Input     |  | Browser Control  |  | Audio Playback Out     | |
       |  | - /dev/uinput    |  | - Chrome CDP     |  | - PipeWire Sink        | |
       |  | - evdev ioctls   |  | - DevTools MCP   |  | - 24kHz -> 48kHz HW    | |
       |  +------------------+  +------------------+  +------------------------+ |
       +-------------------------------------+-----------------------------------+
                                             |
                                             v
       +-------------------------------------------------------------------------+
       |                     MEMORY & PERSISTENCE SUBSYSTEM                      |
       |  [L1: Working RAM Context]   [L2: SQLite WAL Store]  [L3: Vector Embeds]|
       +-------------------------------------------------------------------------+
```

---

## 3. Empirical Verification Findings & Meta-Analysis

During Phase 0, a rigorous audit was conducted on the host machine to challenge and verify initial baseline assumptions. Multiple fatal blindspots and invalid hardware assumptions were uncovered and rectified:

| # | Prior Investigation Claim | Empirical Verification Result | Corrected Architectural Finding |
|:--|:---|:---|:---|
| **1** | **Hardcoded ALSA capture `hw:0,7` (DMIC16kHz)** | `wpctl status` shows active default sink is `87. WI-C100` (Sony Bluetooth) and active source is `95. bluez_input` (Sony mic). | **CRITICAL:** Hardcoding `hw:0,7` would bypass user's active Bluetooth headset! Must route dynamically through PipeWire `@DEFAULT_AUDIO_SOURCE@` / `@DEFAULT_AUDIO_SINK@`. |
| **2** | **Display resolution assumed 1080p (1920x1080)** | `hyprctl monitors -j` shows BOE 0x0A21 on `eDP-1` is native **1366x768 @ 60Hz** (WXGA 16:9). | Screen capture calculations must use native 1366x768 geometry; downsampling to 1024x576 produces negligible savings while blurring code text. |
| **3** | **Virtual input requires root elevation & `sudo pacman -S ydotool`** | `getfacl /dev/uinput` shows POSIX ACL `user:Qelvyrn:rw-`. Python `os.open("/dev/uinput", os.O_WRONLY)` succeeded with fd 10. | **CRITICAL:** No root daemon `ydotoold` or `sudo` required! User `Qelvyrn` has direct user-space `/dev/uinput` access via Linux `evdev` ioctls. |
| **4** | **Push-to-Talk hotkey bound to `SUPER + V`** | `~/.config/hypr/config/binds.lua` line 129 binds `SUPER + V` to Noctalia clipboard panel. | **COLLISION:** `SUPER + V` is occupied. Re-routed Push-to-Talk to unassigned hotkey **`SUPER + G`** (Gemini). |
| **5** | **Live API output sample rate assumed 16 kHz** | Gemini Live API specification confirms output audio is **24,000 Hz Linear PCM** (`audio/pcm;rate=24000`). | Playing 24 kHz audio through a 16 kHz sink causes 1.5x chipmunk speed distortion. Playback pipeline must invoke `paplay --rate=24000 --raw`. |
| **6** | **Live API SDK snippet contained invalid types** | Prior snippet used `types.LiveModality.AUDIO` and `generation_config=GenerateContentConfig`. | Corrected to official `google-genai` 3.2.0 schema: `types.Modality.AUDIO` and `types.LiveConnectConfig`. |
| **7** | **Package management relies on `uv` or system `pip`** | `which uv` and `which pip` fail (exit code 2). Sudo is password-gated. Python bundles `pip 26.2.2` via `ensurepip`. | Isolated virtual environment bootstrapped cleanly via standard `python -m venv` with zero external dependencies. |
| **8** | **Hyprland IPC polled via `.socket.sock`** | Directory listing reveals `$XDG_RUNTIME_DIR/hypr/.../.socket2.sock`. | Replaced polling with zero-latency push event listener on `.socket2.sock` (`activewindow`, `workspace`, `fullscreen`). |
| **9** | **Available RAM claimed to be 3.9 GiB** | Empirical `free -h` shows **4.8 GiB used, only 2.6 GiB available, 7.4 MiB swap active**. | Memory pressure is significantly tighter. Hard ceiling of **<= 400 MB RSS** enforced; heavy local LLMs strictly banned. |

---

## 4. Architecture Specifications Index

All nine technical architecture specifications have been authored and placed under [`docs/`](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/):

1. **[DOC-01: Host Infrastructure & Discovery Baseline](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-01-INFRA.md)**  
   *Hardware profile (Intel Core i3-1215U 2P+4E, 7.4 GiB RAM, 1366x768 display), CachyOS kernel 7.3.4, software stack, PipeWire routing, and `/dev/uinput` POSIX ACL verification.*
2. **[DOC-02: High-Level System Architecture & Agent Topology](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-02-ARCH.md)**  
   *Tri-tier cognitive topology (Local Perception, Reactive Voice Engine, Deliberative Planning Core), IPC protocols, daemon socket, state machine, and process concurrency model.*
3. **[DOC-03: Speech, Voice & Audio Streaming Pipeline Specification](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-03-AUDIO.md)**  
   *PipeWire dynamic ingestion (16 kHz mono S16LE), 24 kHz PCM playback pipeline via `paplay`, Push-to-Talk (`SUPER + G`), Silero VAD v5, acoustic echo cancellation, and verified `google-genai` 3.2.0 Live API schema.*
4. **[DOC-04: Multimodal Vision & Screen Ingestion Architecture](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-04-VISION.md)**  
   *Wayland screen capture via `grim` in-memory pipes (native 1366x768 JPEG), active window geometry cropping (`hyprctl activewindow -j`), V4L2 on-demand webcam capture, token rate budgeting (0.5 FPS), and privacy window masking.*
5. **[DOC-05: Desktop Automation & Hyprland OS Tool Execution Engine](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-05-TOOLS.md)**  
   *Native direct `/dev/uinput` virtual input synthesis (zero-root), Hyprland `.socket2.sock` reactive event bus, Noctalia desktop shell CLI dispatcher, UWSM application lifecycle management, and sandboxed CLI runner.*
6. **[DOC-06: Memory, Context & State Store Architecture](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-06-MEMORY.md)**  
   *Tri-tier memory hierarchy (L1 working RAM context, L2 SQLite 3.48+ WAL episodic store with FTS5, L3 lightweight flat vector index), sliding context compaction, and prompt header pinning.*
7. **[DOC-07: Security, Privacy & Permission Boundary Model](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-07-SECURITY.md)**  
   *Autonomous vs. Human-in-the-Loop (HITL) execution matrix (Tier 0 Read-Only, Tier 1 Benign, Tier 2 Destructive Gate), headless credential sourcing (`0600`), output regex sanitization, and Wayland protocol security compliance.*
8. **[DOC-08: Phased Engineering Implementation Roadmap](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-08-ROADMAP.md)**  
   *Detailed 5-week engineering roadmap: Phase 0 (Complete), Phase 1 (Foundation & IPC), Phase 2 (Audio & Live API), Phase 3 (Multimodal Vision), Phase 4 (Desktop Automation), and Phase 5 (Hardening & Systemd Service).*
9. **[DOC-09: System Risk Assessment & Mitigation Matrix](file:///home/Qelvyrn/.gemini/antigravity/scratch/jarvis/docs/DOC-09-RISKS.md)**  
   *Comprehensive 13-point risk register covering memory pressure, sample-rate mismatches, device hopping, keybinding collisions, destructive command prevention, and detailed contingency plans.*

---

## 5. Summary Risk Register

| Risk ID | Failure Vector | Severity | Verified Mitigation |
|:---|:---|:---|:---|
| **R1** | Host Memory Thrashing (2.6 GiB RAM available) | **HIGH** | Core daemon capped at <= 400 MB RSS. Zero local LLMs. Proactive GC & cache purge at 450 MB. |
| **R2** | Audio Sample Rate Mismatch (16k in vs 24k out) | **HIGH** | Feed incoming 24 kHz PCM directly to `paplay --rate=24000 --raw`. PipeWire handles resampling. |
| **R3** | Audio Device Hopping (Bluetooth vs Laptop Mic) | **HIGH** | Dynamic capture/playback via PipeWire `@DEFAULT_AUDIO_SOURCE@` and `@DEFAULT_AUDIO_SINK@`. |
| **R4** | Hotkey Collision with Noctalia Clipboard | **HIGH** | Re-routed Push-to-Talk to unassigned key combination **`SUPER + G`** in `binds.lua`. |
| **R5** | Display Geometry Mismatch (1366x768 vs 1080p) | **MEDIUM** | Ingest native 1366x768 geometry; eliminate wasteful downsampling that degrades OCR text quality. |
| **R6** | Missing Pip/UV & Sudo Password-Gating | **HIGH** | Bootstrap Python virtual environment via `python -m venv` (bundles `pip 26.2.2`). Zero root needed. |
| **R7** | Acoustic Feedback in Laptop Speaker Mode | **HIGH** | Engage PipeWire `libpipewire-module-echo-cancel` virtual node; bypass when Bluetooth active. |
| **R8** | Display Sleep During Spoken Turns | **MEDIUM** | Call `noctalia msg caffeine-enable` on voice turn start and `caffeine-disable` on turn finish. |
| **R9** | Destructive Command Execution via Hallucination | **HIGH** | Strict Tier 2 HITL gate: destructive actions (`rm`, `git push -f`, `sudo`) require desktop confirmation. |
| **R10**| Orphaned Subprocesses from Shell Tools | **MEDIUM** | GUI apps launched via `uwsm app --`; CLI subprocesses killed via process groups (`os.killpg`). |
| **R11**| Sensitive Data Leakage via Screen Capture | **MEDIUM** | Client-side window metadata blacklist redacts password managers and private browser windows. |
| **R12**| Cloud WebSocket Network Disconnects | **MEDIUM** | Async reconnection with exponential backoff using session resumption tokens (`LiveSessionResume`). |
| **R13**| Python 3.14 Module Deprecations (`audioop`) | **HIGH** | Rely on modern NumPy array operations and standard C structs; ban legacy audio packages. |

---

## 6. Implementation Readiness & Next Steps

Phase 0 discovery and architecture design is **100% complete**:
- The host system has been audited and cataloged.
- The 9 architecture specifications have been authored and verified.
- The risk register and mitigation matrix are fully articulated.
- **Zero implementation code has been written.**

### Recommended Active Workspace
It is recommended that the user set `/home/Qelvyrn/.gemini/antigravity/scratch/jarvis` as the active workspace in Antigravity.

### Next Action: Phase 1 Authorization
Upon human review and formal approval, engineering will proceed immediately with **Phase 1: Environment Setup, IPC Harness & Input Foundation** at `/home/Qelvyrn/.gemini/antigravity/scratch/jarvis`.
