# Document 08: Phased Engineering Implementation Roadmap (DOC-08-ROADMAP)

**Document Reference:** `JARVIS-SPEC-DOC-08`  
**Classification:** Engineering Roadmap & Implementation Plan  
**Target Environment:** CachyOS Linux x86_64, Gemini API Ecosystem  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Phased Strategy

Building an operating-system-level AI assistant requires strict engineering discipline. Attempting to implement voice, vision, and desktop tool execution all at once leads to untraceable regressions, latency spikes, and system instability.

The implementation of Project JARVIS is divided into **six distinct phases (Phase 0 through Phase 5)**. Each phase has concrete deliverables, measurable performance criteria, and a formal exit gate.

```
+-----------------------------------------------------------------------------------+
|                            PHASED IMPLEMENTATION PIPELINE                         |
+-----------------------------------------------------------------------------------+
| [Phase 0: Discovery & Architecture Design] (CURRENT - 100% Complete & Verified)  |
|   - Host discovery, hardware profiling, 9 specs, zero code, pending approval      |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v (Human Gate / Approval Required)
+-----------------------------------------------------------------------------------+
| [Phase 1: Environment Setup, IPC Harness & Input Foundation] (Week 1)             |
|   - Virtual environment bootstrap via `python -m venv` (ensurepip)                |
|   - Direct `/dev/uinput` user-space virtual keyboard/mouse verification           |
|   - Hyprland `.socket2.sock` event listener & Noctalia CLI dispatcher             |
|   - SQLite WAL database schema creation & CLI testing harness                     |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| [Phase 2: Speech Ingestion, VAD & Gemini Live Audio Pipeline] (Week 2)            |
|   - PipeWire `parec` dynamic 16 kHz capture (Sony WI-C100 & Alder Lake DMIC)      |
|   - Silero VAD v5 ONNX CPU integration (< 1% CPU overhead)                        |
|   - `google-genai` 3.2.0 persistent WebSocket Live client (`gemini-3.8-live`)     |
|   - 24 kHz Linear PCM output pipeline (`paplay --rate=24000 --raw`)               |
|   - Sub-500ms voice turn latency & instant barge-in playback termination          |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| [Phase 3: Multimodal Vision & Screen Ingestion Engine] (Week 3)                   |
|   - In-memory `grim` capture pipeline (native 1366x768 @ 75% quality JPEG)        |
|   - Active window geometry cropping (`hyprctl activewindow -j`)                  |
|   - V4L2 on-demand webcam frame extractor via FFmpeg (`/dev/video0`)              |
|   - Privacy window masking filter (password managers / private browsing)         |
|   - Adaptive token rate limiter (0.5 FPS continuous / on-demand default)          |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| [Phase 4: Desktop Automation, Noctalia UI & Tool Execution Broker] (Week 4)       |
|   - UWSM application launcher (`uwsm app --`) with cgroup isolation               |
|   - Noctalia status bar integration (recording indicator, alert pills, caffeine) |
|   - Hyprland window/workspace manipulation tools                                  |
|   - Non-blocking asynchronous tool execution in Gemini Live sessions              |
|   - Deliberative reasoning engine handoff (`gemini-3.8-flash`)                    |
+-----------------------------------------+-----------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------+
| [Phase 5: Hardening, Security HITL, Systemd Daemon & Production Readiness] (Week 5) |
|   - Systemd user service unit (`jarvis.service`) with auto-restart                |
|   - Memory ceiling enforcement (proactive garbage collection > 400 MB RSS)        |
|   - Tier 2 HITL desktop confirmation dialog & vocal confirmation gate             |
|   - End-to-end latency benchmarks and 24-hour conversational stability stress test |
+-----------------------------------------------------------------------------------+
```

---

## 2. Phase 0: Discovery & Architecture Design (COMPLETED)
- **Status:** 100% Complete & Empirically Verified on Host Machine.
- **Deliverables:**
  - Empirical verification of CPU (i3-1215U), RAM (2.6 GiB available), Display (1366x768 @ 60Hz), Audio (PipeWire + WI-C100), and Input (`/dev/uinput` POSIX ACL).
  - Nine comprehensive architecture documents (`DOC-01` through `DOC-09`).
  - System Risk Register with concrete mitigations.
  - Zero implementation code written (awaiting formal user sign-off).

---

## 3. Phase 1: Environment Setup, IPC Harness & Input Foundation (Week 1)

### 3.1 Deliverables
1. **Isolated Runtime Bootstrap:**
   - Initialize virtual environment at `/home/Qelvyrn/.gemini/antigravity/scratch/jarvis/.venv` using `python -m venv`.
   - Install verified dependencies (`google-genai>=3.2.0`, `websockets`, `onnxruntime`, `numpy`).
2. **Direct Virtual Input Module (`jarvis.input.uinput`):**
   - Implement user-space `/dev/uinput` virtual device handler.
   - Verify keystroke injection, Unicode clipboard paste, and cursor motion.
3. **Hyprland Event Bus Listener (`jarvis.ipc.hyprland`):**
   - Connect asynchronous reader to `$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`.
   - Emit structured events on `activewindow`, `workspace`, and `fullscreen`.
4. **Noctalia CLI Dispatcher (`jarvis.ipc.noctalia`):**
   - Implement asynchronous wrapper for `noctalia msg` (volume, brightness, caffeine, alerts).
5. **Persistence Initialization (`jarvis.memory.db`):**
   - Initialize SQLite database under `~/.local/share/jarvis/jarvis_memory.db` with WAL mode and full schemas.
6. **CLI Test Harness (`jarvis test`):**
   - Interactive command-line test script validating IPC responses and input injection.

### 3.2 Exit Gate Criteria
- Virtual keyboard successfully types characters in a focused terminal window without root permissions.
- Hyprland event listener logs window changes with < 5 ms latency.
- Noctalia volume commands successfully alter system audio level.
- SQLite database initializes and executes test write transactions in < 2 ms.

---

## 4. Phase 2: Speech Ingestion, VAD & Gemini Live Audio Pipeline (Week 2)

### 4.1 Deliverables
1. **PipeWire Dynamic Audio Ingestion:**
   - Implement asynchronous `parec` wrapper capturing 16,000 Hz mono S16LE.
   - Verify seamless audio routing when Bluetooth headset connects or disconnects.
2. **Local Silero VAD Integration:**
   - Integrate Silero VAD v5 ONNX engine running on CPU EP.
   - Enforce 600 ms silence detection and send `audio_stream_end=True`.
3. **Gemini Live Client (`google-genai` 3.2.0):**
   - Establish persistent bidirectional WebSocket connection to `gemini-3.8-live`.
   - Implement upstream audio chunk streaming (16 kHz Linear PCM).
4. **24 kHz Audio Playback Pipeline:**
   - Implement asynchronous `paplay --raw --rate=24000 --channels=1 --format=s16le` pipeline.
   - Verify distortion-free playback of Gemini synthesized voice chunks.
5. **Acoustic Echo Cancellation & Barge-In:**
   - WirePlumber echo-cancel filter integration for speaker playback.
   - Implement immediate `SIGTERM` kill on `paplay` upon user speech detection during model output.

### 4.2 Exit Gate Criteria
- Voice round-trip latency (speech stop to audio output start) < 500 ms.
- Audio playback verified at correct 24 kHz pitch without chipmunk distortion.
- User barge-in terminates assistant playback in < 50 ms.
- Continuous dialogue session operates with < 3% host CPU usage.

---

## 5. Phase 3: Multimodal Vision & Screen Ingestion Engine (Week 3)

### 5.1 Deliverables
1. **Zero-Copy Grim Capture Engine:**
   - Implement in-memory JPEG capture at native 1366x768 resolution.
   - Measure capture duration (target < 25 ms).
2. **Active Window Cropping:**
   - Ingest window coordinates from `hyprctl activewindow -j` and pipe cropped frames.
3. **V4L2 Webcam Extractor:**
   - Implement on-demand 720p MJPEG frame grab from `/dev/video0` via FFmpeg.
4. **Privacy Redaction Filter:**
   - Implement regex window blacklist (password managers, incognito windows, sudo terminals).
   - Substitute blanked redaction frames on blacklist hits.
5. **Token Rate Controller:**
   - Implement 0.5 FPS throttling for continuous vision mode with 2-minute safety timeout.

### 5.2 Exit Gate Criteria
- Full-screen capture and active-window capture complete in < 25 ms with zero disk writes.
- Privacy filter reliably redacts password manager windows.
- Gemini Multimodal successfully interprets code syntax and terminal error text from captured frames.

---

## 6. Phase 4: Desktop Automation, Noctalia UI & Tool Broker (Week 4)

### 6.1 Deliverables
1. **UWSM Application Launcher:**
   - Implement application launcher invoking `uwsm app -- <app>`.
   - Verify proper systemd cgroup isolation for launched applications.
2. **Noctalia Status Indicator:**
   - Implement custom top-bar recording badge and alert pills during background tasks.
   - Implement automatic `caffeine-enable` during active speech turns.
3. **Function Calling Tool Broker:**
   - Register function declaration schemas for Hyprland, Noctalia, CLI, and vision tools.
   - Implement non-blocking asynchronous execution (`behavior="NON_BLOCKING"`).
4. **Deliberative Reasoning Handoff:**
   - Connect Tier 3 `gemini-3.8-flash` for multi-step reasoning, large code refactoring, and complex analysis.

### 6.2 Exit Gate Criteria
- Gemini Live successfully triggers desktop operations (workspace switch, volume adjustment, app launch) during voice dialogue.
- Assistant speaks natural spoken fillers while background tools execute concurrently.
- All GUI apps launched cleanly under UWSM systemd units.

---

## 7. Phase 5: Hardening, Security HITL, Systemd Daemon & Production Readiness (Week 5)

### 7.1 Deliverables
1. **Systemd User Service Unit:**
   - Create and install `~/.config/systemd/user/jarvis.service`.
   - Enable auto-start on desktop login with `Restart=on-failure`.
2. **Memory Ceiling Enforcement:**
   - Implement proactive RSS memory watchdog.
   - Trigger garbage collection and buffer purge if process RSS exceeds 400 MB.
3. **Human-in-the-Loop (HITL) Security Gate:**
   - Implement graphical confirmation modal and voice confirmation for all Tier 2 destructive actions (`rm`, `git push -f`, `sudo`).
4. **End-to-End Stress Testing:**
   - 24-hour continuous daemon stability test.
   - Rapid disconnect/reconnect WebSocket resilience test.

### 7.2 Exit Gate Criteria
- Daemon operates continuously under 400 MB RSS ceiling without memory leaks.
- Systemd user service starts cleanly upon login.
- 100% of destructive operations blocked until explicit user confirmation.
- Phase 0 to Phase 5 complete; JARVIS operational in production.
