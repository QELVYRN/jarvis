# Document 09: System Risk Assessment & Mitigation Matrix (DOC-09-RISKS)

**Document Reference:** `JARVIS-SPEC-DOC-09`  
**Classification:** Risk Register, Threat Modeling & Mitigation Matrix  
**Target Environment:** CachyOS Linux, Hyprland, Gemini Live / Flash APIs  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Risk Management Methodology

Project JARVIS operates at the intersection of local operating system primitives and cloud artificial intelligence models. System failures in this environment can range from minor cosmetic glitches (e.g. garbled audio) to catastrophic data loss (e.g. unintended recursive deletion of user files) or system lockups (out-of-memory kernel panics).

This document establishes the **Comprehensive System Risk Register**. Every identified failure mode has been analyzed under empirical host conditions, assigned an objective severity level, and paired with a verified, deterministic engineering mitigation.

---

## 2. Comprehensive System Risk Register

| # | Risk Description | Severity | Impact Area | Empirical Cause / Root Vector | Mitigation & Contingency Strategy |
|:--|:---|:---|:---|:---|:---|
| **R1** | **Host Memory Pressure & Thrashing** | **HIGH** | System Stability | Empirical `free -h` shows only 2.6 GiB available RAM. Heavy local processes trigger zram compression and OOM freezes. | Hard daemon ceiling of **<= 400 MB RSS**. Zero local LLMs. Proactive GC and cache purge triggered if RSS crosses 450 MB. |
| **R2** | **Audio Sample-Rate Mismatch & Distortion** | **HIGH** | Audio Fidelity | Gemini Live outputs 24 kHz Linear PCM; feeding this into a 16 kHz configured player results in 1.5x chipmunk speech. | Direct playback via `paplay --raw --rate=24000 --channels=1 --format=s16le`. PipeWire performs AVX2 SIMD resampling to 48 kHz hardware sink. |
| **R3** | **Audio Endpoint Hopping (Bluetooth vs. DMIC)** | **HIGH** | Voice Ingestion | User actively uses Sony WI-C100 Bluetooth headset. Hardcoding ALSA card `hw:0,7` bypasses headset mic and speakers. | Dynamic routing through PipeWire's `@DEFAULT_AUDIO_SOURCE@` and `@DEFAULT_AUDIO_SINK@`. Subprocess handles reconnects automatically. |
| **R4** | **Keybinding Collision with Noctalia Shell** | **HIGH** | Desktop UX | Proposed `SUPER + V` hotkey conflicts with Noctalia's native clipboard manager bind in `binds.lua`. | Re-route Push-to-Talk to unassigned key combination **`SUPER + G`** (Gemini) in `~/.config/hypr/config/binds.lua`. |
| **R5** | **Display Geometry Assumption Mismatch** | **MEDIUM** | Vision Accuracy | Assumed 1080p display is actually native 1366x768 @ 60Hz. Downscaling to 1024x576 blurs terminal code. | Dynamic screen geometry query via `hyprctl monitors -j`. Capture native 1366x768 JPEG at 75% quality (~135 KB). |
| **R6** | **Missing Pip/UV & Password-Gated Sudo** | **HIGH** | Deployment | `which uv` and `which pip` fail. `pacman` requires sudo password. System tools cannot be installed via root. | Bootstrap virtual environment using standard `python -m venv` (which bundles pip 26.2.2). Zero reliance on pacman or root sudo. |
| **R7** | **Acoustic Feedback in Laptop Speaker Mode** | **HIGH** | Audio Quality | Synthesized voice from internal laptop speakers re-enters microphone array, creating echo feedback loops. | Engage PipeWire `libpipewire-module-echo-cancel` virtual node during speaker mode. Automatic bypass when Bluetooth headset is active. |
| **R8** | **Display Sleep / Idle Dimming Mid-Conversation** | **MEDIUM** | Continuity | Hypridle dims screen and engages DPMS sleep during long conversational turns or audio playback. | Call `noctalia msg caffeine-enable` when voice session initiates; call `noctalia msg caffeine-disable` on session close. |
| **R9** | **Destructive Command Execution via Hallucination** | **HIGH** | System Integrity | Model hallucination or prompt injection triggers destructive shell commands (`rm`, `git push -f`, `dd`). | Strict **Tier 2 HITL Boundary**: Destructive shell operations require visual desktop confirmation modal and explicit user approval. |
| **R10**| **Orphaned / Zombie Subprocesses from Tool Shells** | **MEDIUM** | OS Resources | Background subprocesses without process groups linger after parent timeout, consuming CPU/RAM. | Launch GUI applications via `uwsm app --`. CLI subprocesses wrapped in unique process groups (`os.setsid`) and killed via `os.killpg()`. |
| **R11**| **Sensitive Data Leakage via Screen Ingestion** | **MEDIUM** | Privacy | Vision frame capture transmits visible passwords, banking portals, or confidential tokens to cloud API. | Client-side window metadata blacklist (`1password`, `bitwarden`, `keepassxc`, `sudo` terminals) substitutes blanked redacted frame. |
| **R12**| **WebSocket Network Disconnects & Session Drops** | **MEDIUM** | Availability | Transient Wi-Fi drop interrupts bidirectional Gemini Live session. | Async reconnection handler with exponential backoff (1s, 2s, 4s, max 10s) using session resumption tokens. Audible reconnection chime. |
| **R13**| **Python 3.14 Module Deprecations (`audioop`)** | **HIGH** | Audio Pipeline | Python 3.14.10 permanently removed the standard library `audioop` module. Older audio scripts will crash immediately. | Use modern NumPy array operations and standard C `struct.pack`/`unpack` for all PCM sample transformations. |

---

## 3. Detailed Risk Deep-Dives & Contingency Plans

### 3.1 Deep Dive: Risk R1 (Host Memory Pressure & Thrashing)
- **Vulnerability Analysis:** CachyOS utilizes `zram` swap. While fast, zram compresses physical RAM. If RAM usage exceeds 90%, zram compression consumes significant CPU, causing mouse stutter and audio buffer underruns in PipeWire.
- **Architectural Safeguards:**
  1. The core JARVIS daemon process RSS is actively monitored via `/proc/self/statm` on every 10th event loop turn.
  2. If process RSS exceeds **400 MB**, memory compaction engages:
     - Clear in-memory frame cache.
     - Release SQLite memory pools via `sqlite3_db_release_memory()`.
     - Invoke `gc.collect()`.
     - Issue `ctypes.CDLL("libc.so.6").malloc_trim(0)` to force glibc arena release back to the Linux kernel.
  3. Under no circumstances will local 7B+ LLM inference engines (Ollama, vLLM, llama.cpp) be executed on this host.

### 3.2 Deep Dive: Risk R9 (Destructive Command Execution via Hallucination / Injection)
- **Vulnerability Analysis:** When analyzing untrusted web pages, GitHub repositories, or error messages, adversarial instructions embedded in text could trick the model into executing malicious bash scripts.
- **Architectural Safeguards:**
  1. **Deterministic Filter List:** Commands containing destructive patterns (`rm`, `mkfs`, `dd`, `fdisk`, `shred`, `truncate`, `git reset --hard`, `git push --force`, `systemctl stop`, `reboot`, `shutdown`, `chmod -R 777`) are hard-coded into the Tier 2 security category.
  2. **Mandatory Out-of-Band Gate:** The command execution broker cannot bypass the gate via model instruction. A confirmation token must be received directly from the local user via GUI modal or vocal confirmation before `subprocess.exec` is invoked.

### 3.3 Deep Dive: Risk R7 (Acoustic Feedback & Self-Triggering Loops)
- **Vulnerability Analysis:** When playing synthesized speech at 24 kHz through laptop speakers, the onboard microphone array captures the speaker sound waves. Without proper cancellation, the speech enters the Live API WebSocket, causing the model to hear itself, hallucinate responses to its own words, or interrupt its own output.
- **Architectural Safeguards:**
  1. **Hardware Sensing:** On startup and device change events, query `wpctl status` to detect whether audio sink is a headset or speakers.
  2. **Filter-Chain Routing:** If speakers are active, ingest audio from the PipeWire echo-cancelled virtual source (`libpipewire-module-echo-cancel`), which compares the sink monitor channel with the source channel and phase-cancels identical audio in real-time.
  3. **Half-Duplex Fallback Mode:** In environments where echo-cancel fails, JARVIS temporarily mutes microphone chunk ingestion during active `SPEAKING` state, un-muting upon playback completion.

---

## 4. Risk Governance & Continuous Monitoring

During Phase 1 through Phase 5 implementation:
1. Every risk mitigation listed above is tied directly to a specific automated unit test or integration test harness.
2. The SQLite `action_audit` table tracks all security-tier elevations, tool timeouts, and memory trim events.
3. If any architectural invariant is breached (e.g. RSS memory exceeding 450 MB or an unhandled keybinding collision occurring), the daemon logs an emergency error to `systemd` journal and transitions to safe read-only fallback mode.
