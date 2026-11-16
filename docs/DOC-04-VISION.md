# Document 04: Multimodal Vision & Screen Ingestion Architecture (DOC-04-VISION)

**Document Reference:** `JARVIS-SPEC-DOC-04`  
**Classification:** Multimodal Perception & Vision Architecture  
**Target Environment:** Hyprland 0.57.2 Wayland, Grim, V4L2, Gemini Multimodal  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Wayland Ingestion Constraints

Desktop vision enables JARVIS to inspect compile errors, read web pages, debug terminal logs, and navigate graphical user interfaces. Under the modern Wayland security architecture, direct unconstrained screen scraping (such as X11 `scrot` or reading `/dev/fb0`) is strictly prohibited by compositor isolation boundaries.

To provide low-latency visual perception without violating Wayland security protocols or degrading system performance, JARVIS leverages native Hyprland protocols and standard Wayland utilities (`grim`) piped directly into in-memory byte streams.

---

## 2. Display Geometry & Native Resolution Strategy

### 2.1 Empirical Display Profile
Host verification via `hyprctl monitors -j` disproved standard 1080p assumptions:
- **Display Output:** `eDP-1` (Internal BOE 0x0A21 panel).
- **Native Resolution:** **1366x768 pixels** (WXGA 16:9).
- **Refresh Rate:** 59.994 Hz.
- **Scale Factor:** 1.0 (unscaled, 1:1 pixel grid).

### 2.2 Why Downsampling is Counterproductive
- A native 1366x768 frame contains **1,049,088 pixels** (~1.05 MP).
- When encoded as JPEG with 75% quality, the compressed payload averages only **~135 KB**.
- Artificial downscaling to 1024x576 saves fewer than 30 KB of network transfer, but introduces bilinear interpolation blur that severely degrades Gemini's optical character recognition (OCR) on 9pt terminal fonts and code syntax.
- **Specification:** Full-display captures are ingested at native **1366x768** resolution.

---

## 3. Screen Capture Execution Pipeline

### 3.1 Full-Display Zero-Disk Pipeline
JARVIS executes `grim` with standard output directed into an asynchronous Python pipe. No temporary files are written to physical NVMe storage, preserving SSD lifespan and reducing capture latency to **~18 ms**:

```bash
grim -t jpeg -q 75 -
```

```python
import asyncio
from google.genai import types

async def capture_fullscreen_jpeg() -> bytes:
    """Captures full screen directly into memory buffer."""
    process = await asyncio.create_subprocess_exec(
        "grim", "-t", "jpeg", "-q", "75", "-",
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.DEVNULL
    )
    jpeg_bytes, _ = await process.communicate()
    return jpeg_bytes
```

### 3.2 Active-Window Targeted Ingestion
When the user asks questions regarding a specific application (e.g. *"Why is this build failing?"* or *"Summarize this document"*), capturing the full desktop introduces distracting wallpaper, side panels, and unrelated windows.

JARVIS extracts the active window's precise geometry via Hyprland IPC:
1. Query active window: `hyprctl activewindow -j`
2. Parse JSON response:
   ```json
   {
     "at": [12, 48],
     "size": [1342, 708],
     "class": "kitty",
     "title": "nvim src/main.rs"
   }
   ```
3. Pass bounding geometry to `grim -g`:
   ```bash
   grim -g "12,48 1342x708" -t jpeg -q 80 -
   ```
4. Payload size drops to **~75–110 KB**, while focusing 100% of the multimodal vision tokens on the relevant context.

```python
import json

async def capture_active_window_jpeg() -> tuple[bytes, dict]:
    """Captures only the focused Wayland window."""
    # 1. Query geometry
    proc = await asyncio.create_subprocess_exec(
        "hyprctl", "activewindow", "-j",
        stdout=asyncio.subprocess.PIPE
    )
    out, _ = await proc.communicate()
    meta = json.loads(out)
    
    x, y = meta["at"]
    w, h = meta["size"]
    geo_str = f"{x},{y} {w}x{h}"
    
    # 2. Crop capture
    crop_proc = await asyncio.create_subprocess_exec(
        "grim", "-g", geo_str, "-t", "jpeg", "-q", "80", "-",
        stdout=asyncio.subprocess.PIPE
    )
    jpeg_bytes, _ = await crop_proc.communicate()
    return jpeg_bytes, meta
```

### 3.3 Interactive Region & Annotation Integration
When user-guided selection is desired, JARVIS invokes Noctalia's built-in region selector:
- `noctalia msg screenshot-region`
- `noctalia msg annotate`
This triggers the user-facing Wayland overlay without requiring external tools like `slurp` to manage cross-display seat handles.

---

## 4. Integrated HD Webcam Ingestion Subsystem

### 4.1 Device Profile
- **Device Paths:** `/dev/video0` and `/dev/video1`.
- **Identity:** `Integrated_Webcam_HD: Integrate` (V4L2 driver).
- **Supported Video Modes:**
  - MJPEG: 1280x720 @ 30 fps, 640x480 @ 30 fps.
  - YUYV 4:2:2: 640x480 @ 30 fps, 1280x720 @ 10 fps.

### 4.2 On-Demand Single-Frame Capture
To prevent privacy intrusions and avoid locking the camera device node from other applications (e.g. Zoom, Discord), persistent camera streaming is **strictly prohibited**.

JARVIS extracts single frames on demand using hardware-accelerated FFmpeg:
```bash
ffmpeg -f v4l2 -input_format mjpeg -video_size 1280x720 -i /dev/video0 -vframes 1 -f image2pipe -vcodec copy -
```
- Capture latency: **~45 ms**.
- Output: Direct JPEG bytes extracted from the camera's hardware MJPEG encoder with zero CPU re-encoding overhead.

---

## 5. Multimodal Token Ingestion Budget & Adaptive Frame Rates

Streaming high-framerate video to cloud multimodal models causes massive token consumption and will quickly exhaust API rate limits and quotas.

```
       +-------------------------------------------------------------+
       |                  MULTIMODAL INGESTION MODES                 |
       +-------------------------------------------------------------+
       | Mode 1: Static / On-Demand (Default)                        |
       | - 1 frame captured only when user asks a vision question    |
       | - Token Cost: ~258 tokens per capture                       |
       | - Zero background bandwidth                                 |
       +-------------------------------------------------------------+
       | Mode 2: Continuous Desktop Stream (Active Troubleshooting)  |
       | - 1 frame every 2.0 seconds (0.50 FPS)                      |
       | - Token Cost: ~7,740 tokens / minute                        |
       | - Automatic timeout: Stops after 2 minutes of silence       |
       +-------------------------------------------------------------+
```

### 5.1 Token Budget Rules:
1. **On-Demand Default:** By default, vision is **dormant**. When the user asks *"What is on my screen?"* or *"Look at this"*, a single high-quality frame is captured and injected.
2. **Continuous Assistance Mode:** If the user initiates continuous desktop co-piloting (e.g., *"Help me through this installation step-by-step"*), JARVIS enables streaming at **0.5 FPS (1 frame every 2 seconds)**.
3. **Safety Timeout:** Continuous streaming automatically shuts down if no voice dialogue occurs for 120 seconds.

---

## 6. Privacy Guardrails & Window Blacklisting

Operating an agent with visual desktop access introduces severe privacy risks regarding passwords, credit card numbers, and confidential communications.

### 6.1 Automated Window Class Filtering
Before any screen capture is forwarded to the Gemini API or saved to episodic memory logs, the active window's metadata (`class` and `title`) is evaluated against the **Strict Privacy Blacklist**:

| Blacklist Pattern | Target Applications / Activities |
|:---|:---|
| `*1password*`, `*bitwarden*`, `*keepassxc*` | Password Managers |
| `*private*`, `*incognito*` | Private Browser Windows |
| `*gpg*`, `*pass*`, `*seahorse*` | Cryptographic Key Stores |
| Terminal window running `sudo *` | Elevated Root Shell Sessions |
| Application containing `*bank*`, `*paypal*` | Financial Portals |

### 6.2 Blacklist Action
If a match occurs:
1. Screen capture is **instantly aborted**.
2. A synthetic, blanked SVG/JPEG image containing the text `[PROTECTED WINDOW: REDACTED FOR PRIVACY]` is substituted.
3. An audit log entry is written: `PRIVACY_FILTER_ENGAGED: class=<class>, title=<title>`.
4. JARVIS vocalizes: *"I've redacted visual capture because a sensitive application is focused."*

---

## 7. Multimodal Ingestion Verification Table

| Metric / Parameter | Value / Implementation |
|:---|:---|
| **Capture Utility** | `grim` 1.5.0 via Wayland protocol. |
| **Full Resolution** | Native 1366x768 pixels @ 1.0 scale (BOE 0x0A21). |
| **Crop Method** | `hyprctl activewindow -j` -> `grim -g`. |
| **Format & Quality** | JPEG at 75% quality (~135 KB full / ~85 KB active window). |
| **In-Memory Pipe** | Subprocess pipe to Python memory; 0 disk writes. |
| **Webcam Format** | MJPEG 1280x720 single-frame on-demand extraction via FFmpeg. |
| **Continuous Rate**| 0.5 FPS (1 frame / 2s), capped at 2-minute timeout. |
| **Privacy Filter** | Client-side metadata pre-filtering before cloud transmission. |
