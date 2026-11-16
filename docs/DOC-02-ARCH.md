# Document 02: High-Level System Architecture & Agent Topology (DOC-02-ARCH)

**Document Reference:** `JARVIS-SPEC-DOC-02`  
**Classification:** System Architecture & Topology Specification  
**Target Environment:** CachyOS Linux x86_64, Hyprland 0.57.2, Gemini 3.8  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Architectural Philosophy

Project JARVIS is structured around a **three-tier asymmetric cognitive topology**. Modern LLM operating assistants frequently fail because they attempt to use a single monolithic model for all tasks, resulting in high latency for basic voice conversation or severe local resource exhaustion.

JARVIS resolves this by cleanly separating:
1. **Local Real-Time Perception (Host OS):** Sub-millisecond event filtering, audio buffering, VAD energy gating, and window tracking running natively on CachyOS.
2. **Reactive Conversational Engine (`gemini-3.8-live`):** Continuous bidirectional WebSocket streaming providing immediate spoken dialogue, non-blocking background tool triggers, and voice interruption.
3. **Deliberative Reasoning Core (`gemini-3.8-flash`):** Complex multi-step reasoning, code refactoring, system diagnostics, and calibrated safety decisions invoked on demand.

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

## 2. Multi-Tier Cognitive Topology

### 2.1 Tier 1: Local Perception & Event Engine (Host OS)
- **Execution Target:** Local CPU (P-cores 0–3 for audio/events, E-cores 4–7 for storage).
- **Resource Footprint:** < 2.5% CPU utilization when idle, < 45 MB RAM.
- **Responsibilities:**
  - Ingests raw audio via PipeWire `parec` at 16,000 Hz S16LE.
  - Executes low-latency energy and ONNX-based Voice Activity Detection (Silero VAD v5).
  - Listens continuously to Hyprland's native event socket (`.socket2.sock`) for active window title changes, workspace shifts, and application launch events.
  - Takes zero-copy in-memory Wayland screen snapshots via `grim` into `/dev/shm`.
  - Captures Push-to-Talk hotkey triggers (`SUPER + G`).

### 2.2 Tier 2: Reactive Real-Time Voice Engine (`gemini-3.8-live`)
- **Execution Target:** Google Gemini Cloud via persistent WebSocket.
- **Protocol:** `google-genai` SDK 3.2.0 `client.aio.live.connect()`.
- **Latency Profile:** ~250–450 ms voice-to-voice round-trip.
- **Responsibilities:**
  - Streams upstream 16 kHz Linear PCM audio chunks in real-time.
  - Receives downstream 24 kHz Linear PCM audio chunks and streams them directly to `paplay`.
  - Declares non-blocking asynchronous tools (`behavior="NON_BLOCKING"`). When JARVIS triggers a desktop operation (e.g., launching an app, setting volume, querying a window), the model speaks a natural conversational filler (e.g., *"Opening Spotify now..."*) while the action executes concurrently in the background.
  - Upgrades dynamically to `gemini-3.8-live-extended-thinking` when complex conversational reasoning or background problem-solving is requested, tracking session state via `interaction_status`.

### 2.3 Tier 3: Deliberative Planning Core (`gemini-3.8-flash`)
- **Execution Target:** Google Gemini Cloud via REST / gRPC API (`client.aio.models.generate_content`).
- **Context Capacity:** 1,048,576 tokens.
- **Responsibilities:**
  - Triggered by Tier 2 whenever an action exceeds reactive capabilities: multi-file code editing, system diagnostics, complex shell script generation, or high-risk administrative operations.
  - Executes with calibrated thinking and structured tool orchestration.
  - Formulates verified plans, evaluates potential failure modes, and yields structured results back to Tier 2 for vocal summarization to the user.

---

## 3. Inter-Process Communication (IPC) Protocol

### 3.1 JARVIS Core Daemon Socket (`jarvis.sock`)
- **Location:** `$XDG_RUNTIME_DIR/jarvis/jarvis.sock` (typically `/run/user/1000/jarvis/jarvis.sock`).
- **Transport:** UNIX domain stream socket.
- **Wire Format:** Length-prefixed JSON-RPC 2.0 messages.
- **Endpoints:**
  - `ptt_toggle`: Toggles or asserts the active Push-to-Talk listening state.
  - `inject_context`: Pushes arbitrary desktop events into the L1 working context.
  - `status_query`: Returns daemon health, active model, token count, and RSS memory usage.
  - `interrupt`: Triggers immediate voice playback interruption and turn cancellation.

### 3.2 Hyprland Real-Time Event Ingress (`.socket2.sock`)
Hyprland exposes an asynchronous, event-driven socket at:
`$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`

Rather than polling `hyprctl` in an expensive loop, JARVIS maintains a persistent `asyncio` line-delimited stream listener. The stream emits immediate notifications:
- `activewindow>>class,title`
- `workspace>>workspace_name`
- `fullscreen>>0|1`
- `openwindow>>window_address,workspace_name,class,title`
- `closewindow>>window_address`

This allows JARVIS to maintain instantaneous awareness of the user's active application (e.g. knowing when the user enters Discord, VS Code, or a terminal running `nvim`) with zero CPU polling overhead.

### 3.3 Hyprland Command Egress (`.socket.sock` / `hyprctl`)
For window manipulation, JARVIS writes commands to Hyprland's request socket or executes `hyprctl dispatch`:
- Format: `dispatch <dispatcher_name> <arguments>`
- Commands: `workspace`, `focuswindow`, `movetoworkspace`, `killactive`, `fullscreen`.

### 3.4 Noctalia Desktop Shell IPC
Noctalia provides a first-class CLI command bridge: `/usr/bin/noctalia msg <subcommand>`.
JARVIS interacts with Noctalia without monkey-patching or screen scraping:
- `volume-set <val>`, `volume-mute`, `volume-down`, `volume-up`.
- `brightness-set <val>`, `brightness-down`, `brightness-up`.
- `caffeine-enable` / `caffeine-disable` (prevents screen dimming/DPMS sleep during voice interactions).
- `workspace-alert-add <workspace>` (places an agent status badge on the desktop bar).
- `screenshot-region`, `annotate` (visual overlays).

---

## 4. State Machine & Session Lifecycle

The JARVIS runtime operates as a deterministic finite-state automaton (FSM):

```
                      +-------------------+
                      |       IDLE        |<--------------------+
                      +---------+---------+                     |
                                |                               |
                 PTT Press / VAD Voice Detected                 |
                                |                               |
                                v                               |
                      +-------------------+                     |
                      |     LISTENING     |                     |
                      +---------+---------+                     |
                                |                               |
                       Silence / PTT Release                    |
                                |                               |
                                v                               |
                      +-------------------+                     |
                      |     THINKING      |                     |
                      +---------+---------+                     |
                                |                               |
                      Model Generates Audio                     |
                                |                               |
                                v                               |
         +------------+  Audio Output     +-------------------+ |
         |INTERRUPTED |<------------------|     SPEAKING      | |
         +-----+------+   User Barge-In   +---------+---------+ |
               |                                    |           |
               v                          Tool Call |           |
         (Clear Buffer)                   Triggered |           |
               |                                    v           |
               |                          +-------------------+ |
               +------------------------->|     EXECUTING     |-+
                                          +-------------------+
```

### State Definitions:
1. **IDLE:**
   - Microphone stream read buffer is discarded or gated by VAD.
   - Hyprland `.socket2.sock` listener remains active, updating working context.
   - CPU utilization < 1.0%.
2. **LISTENING:**
   - Triggered by `SUPER + G` hotkey press or local VAD energy trigger.
   - Audio chunks (16 kHz PCM) are forwarded to the active Gemini Live WebSocket.
   - Noctalia recording indicator displayed on screen.
3. **THINKING:**
   - End of turn detected (`audio_stream_end=True` sent).
   - If using `gemini-3.8-live-extended-thinking`, background reasoning frames and conversational fillers may be received.
4. **SPEAKING:**
   - 24 kHz PCM audio chunks streamed to `paplay`.
   - Ingress audio continues to be monitored for user barge-in.
5. **EXECUTING:**
   - Tool execution triggered asynchronously (`behavior="NON_BLOCKING"`).
   - Desktop command dispatches to Hyprland, Noctalia, or CLI subshell.
   - Tool response returned to Live session.
6. **INTERRUPTED:**
   - User speech detected while model is speaking.
   - `paplay` subprocess receives `SIGTERM`; playback buffer is instantly dumped.
   - Gemini Live session receives client interruption signal; state resets to LISTENING.

---

## 5. Process & Thread Concurrency Model

```
+-----------------------------------------------------------------------------------+
|                           JARVIS Main Process (Python 3.14)                       |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  | asyncio Main Event Loop (Affinity: P-Cores 0-3)                             |  |
|  |                                                                             |  |
|  |  +-------------------+  +--------------------+  +-------------------------+ |  |
|  |  | Audio In Stream   |  | Audio Out Pipeline |  | Gemini Live WebSocket   | |  |
|  |  | (parec pipe)      |  | (paplay pipe)      |  | (aio.live.connect)      | |  |
|  |  +-------------------+  +--------------------+  +-------------------------+ |  |
|  |  +-------------------+  +--------------------+  +-------------------------+ |  |
|  |  | Hyprland socket2  |  | Noctalia IPC Bus   |  | Unix Domain Socket      | |  |
|  |  | Listener          |  | Dispatcher         |  | Server (jarvis.sock)    | |  |
|  |  +-------------------+  +--------------------+  +-------------------------+ |  |
|  +-----------------------------------------------------------------------------+  |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  | ThreadPoolExecutor / ProcessPool (Affinity: E-Cores 4-7)                    |  |
|  |                                                                             |  |
|  |  [Worker 1: SQLite WAL Log]   [Worker 2: Disk I/O]   [Worker 3: Screen Enc] |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

- **Asyncio Single-Threaded Reactor:** Drives all network sockets, UNIX pipes, and process IPC without GIL lockups or thread synchronization overhead.
- **Worker Thread Pool:** Offloads synchronous operations (disk writes, SQLite queries, image conversions) to background threads pinned to E-cores, ensuring the voice audio loop maintains zero jitter.
