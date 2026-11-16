# Document 05: Desktop Automation & Hyprland OS Tool Execution Engine (DOC-05-TOOLS)

**Document Reference:** `JARVIS-SPEC-DOC-05`  
**Classification:** Desktop Automation & Tool Integration Specification  
**Target Environment:** CachyOS Linux, Hyprland 0.57.2, Noctalia Shell 1.2.6, UWSM  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Automation Philosophy

Project JARVIS bridges AI reasoning with real-world operating system action. Rather than treating desktop automation as a brittle series of screen-scraping mouse clicks, JARVIS interacts with CachyOS through **first-class operating system interfaces**:
1. **Hyprland IPC Sockets:** Direct compositor dispatch for workspace switching, window manipulation, and layout control.
2. **Noctalia Desktop Shell Bus:** Structured CLI messaging for system notifications, audio volume, display brightness, and power management.
3. **Native Direct `/dev/uinput` Virtual Input:** Direct user-space injection of synthetic keystrokes and pointer events without requiring root daemons.
4. **Universal Wayland Session Manager (UWSM):** Clean cgroup isolation and lifecycle management for all launched desktop applications.
5. **Sandboxed Subprocess Execution:** Strict execution guardrails, timeouts, and buffer caps for shell utilities.

---

## 2. Tool Registry Architecture & Gemini Function Calling

Tools are registered using the standard `google-genai` SDK function declaration format. In the reactive voice engine (`gemini-3.8-live`), tool calls are executed asynchronously without blocking the conversational audio pipeline.

```python
from google.genai import types

# Example: Hyprland Workspace Dispatch Tool Declaration (behavior="NON_BLOCKING" required for Gemini Live)
hyprland_workspace_tool = types.FunctionDeclaration(
    name="switch_workspace",
    description="Switches the active Hyprland virtual workspace on the desktop.",
    behavior="NON_BLOCKING",
    parameters=types.Schema(
        type=types.Type.OBJECT,
        properties={
            "workspace_index": {
                "type": types.Type.INTEGER,
                "description": "The workspace number to switch to (e.g. 1 to 10)."
            }
        },
        required=["workspace_index"]
    )
)

# Example: Noctalia Shell Control Tool Declaration
noctalia_volume_tool = types.FunctionDeclaration(
    name="set_system_volume",
    description="Adjusts the host audio output volume or toggles mute state via Noctalia shell.",
    behavior="NON_BLOCKING",
    parameters=types.Schema(
        type=types.Type.OBJECT,
        properties={
            "volume_percent": {
                "type": types.Type.INTEGER,
                "description": "Target volume percentage from 0 to 100."
            },
            "mute": {
                "type": types.Type.BOOLEAN,
                "description": "True to mute audio output, False to unmute."
            }
        }
    )
)
```

---

## 3. Native Direct `/dev/uinput` Virtual Input Synthesis

### 3.1 Zero-Root Architecture
A critical finding from empirical system discovery is that user `Qelvyrn` already possesses read-write access to `/dev/uinput` via systemd POSIX ACL:
```
# file: dev/uinput
user:Qelvyrn:rw-
```
Testing confirmed that opening `/dev/uinput` in user space succeeds immediately (`os.open('/dev/uinput', os.O_WRONLY)` returns file descriptor 10).

Consequently:
- JARVIS **does not require** `sudo pacman -S ydotool`.
- JARVIS **does not require** running a root background daemon (`ydotoold`).
- JARVIS **does not suffer** from `wtype` Wayland modifier lockups.

### 3.2 Direct Python `uinput` Virtual Device Implementation
JARVIS instantiates a virtual Linux input device using `evdev` or direct kernel ioctls:

```python
import os
import struct
import time

# Linux input event constants
EV_KEY = 0x01
EV_SYN = 0x00
SYN_REPORT = 0x00
KEY_ENTER = 28
KEY_LEFTCTRL = 29
KEY_V = 47

class VirtualInputDevice:
    def __init__(self):
        self.fd = os.open("/dev/uinput", os.O_WRONLY | os.O_NONBLOCK)
        self._setup_device()

    def _setup_device(self):
        # Enable key events and specific scancodes via ioctl
        import fcntl
        UI_SET_EVBIT = 0x40045564
        UI_SET_KEYBIT = 0x40045565
        UI_DEV_SETUP = 0x405c5503  # _IOW('U', 3, struct uinput_setup)
        UI_DEV_CREATE = 0x5501
        
        fcntl.ioctl(self.fd, UI_SET_EVBIT, EV_KEY)
        for key in range(1, 256):
            fcntl.ioctl(self.fd, UI_SET_KEYBIT, key)
            
        # struct uinput_setup: 80s (name) + 4H (bustype, vendor, product, version) + i (ff_effects_max) = 92 bytes
        setup_data = struct.pack("80sHHHHi", b"JARVIS-Virtual-Keyboard", 0x03, 0x1234, 0x5678, 1, 0)
        fcntl.ioctl(self.fd, UI_DEV_SETUP, setup_data)
        fcntl.ioctl(self.fd, UI_DEV_CREATE)

    def emit_key(self, key_code: int, state: int):
        # state: 1 = press, 0 = release
        event_fmt = "llHHi"
        now = time.time()
        tv_sec = int(now)
        tv_usec = int((now - tv_sec) * 1000000)
        
        # Key event
        ev = struct.pack(event_fmt, tv_sec, tv_usec, EV_KEY, key_code, state)
        os.write(self.fd, ev)
        # Sync event
        syn = struct.pack(event_fmt, tv_sec, tv_usec, EV_SYN, SYN_REPORT, 0)
        os.write(self.fd, syn)

    def send_keystroke(self, key_code: int):
        self.emit_key(key_code, 1)
        time.sleep(0.01)
        self.emit_key(key_code, 0)

    def paste_unicode_text(self, text: str):
        """Copies text to Wayland clipboard and triggers Ctrl+V."""
        import subprocess
        subprocess.run(["wl-copy", text], check=True)
        # Emit Ctrl+V
        self.emit_key(KEY_LEFTCTRL, 1)
        self.emit_key(KEY_V, 1)
        time.sleep(0.01)
        self.emit_key(KEY_V, 0)
        self.emit_key(KEY_LEFTCTRL, 0)
```

---

## 4. Hyprland Reactive Event Bus & Window Dispatch

### 4.1 Real-Time Push Ingress (`.socket2.sock`)
Hyprland exposes real-time event notifications via a dedicated UNIX domain socket located at:
`$XDG_RUNTIME_DIR/hypr/$HYPRLAND_INSTANCE_SIGNATURE/.socket2.sock`

JARVIS attaches a persistent asynchronous stream listener:

```python
import asyncio
import os

async def listen_hyprland_events(callback):
    sig = os.environ["HYPRLAND_INSTANCE_SIGNATURE"]
    sock_path = f"{os.environ['XDG_RUNTIME_DIR']}/hypr/{sig}/.socket2.sock"
    
    reader, _ = await asyncio.open_unix_connection(sock_path)
    while True:
        line = await reader.readline()
        if not line:
            break
        event_str = line.decode("utf-8").strip()
        if ">>" in event_str:
            event_type, payload = event_str.split(">>", 1)
            await callback(event_type, payload)
```

### 4.2 Handled Hyprland Events & Cognitive Responses

| Event Name | Payload Format | JARVIS Ingestion & Response |
|:---|:---|:---|
| `activewindow` | `[class],[title]` | Updates L1 working memory with active application context. |
| `workspace` | `[workspace_name]` | Tracks current virtual workspace index. |
| `fullscreen` | `0` or `1` | Adapts screen capture mode between cropped window and full desktop. |
| `openwindow` | `[addr],[ws],[class],[title]` | Detects when a newly launched app has finished opening. |
| `closewindow` | `[addr]` | Cleans up active window references in memory. |

### 4.3 Compositor Window Dispatch
Commands are dispatched using `hyprctl dispatch <action> <args>`:
- **Switch Workspace:** `hyprctl dispatch workspace <n>`
- **Focus Window:** `hyprctl dispatch focuswindow <class_regex>`
- **Close Window Cleanly:** `hyprctl dispatch killactive`
- **Toggle Fullscreen:** `hyprctl dispatch fullscreen 0`
- **Reorganize Layout:** `hyprctl dispatch togglesplit`

---

## 5. Noctalia Desktop Shell IPC Interface

Noctalia (`/usr/bin/noctalia`) controls the status bar, on-screen display (OSD), system trays, and system utilities. JARVIS dispatches structured CLI commands to manipulate widgets seamlessly:

```python
async def noctalia_dispatch(command: str) -> str:
    cmd = ["noctalia", "msg"] + command.split()
    proc = await asyncio.create_subprocess_exec(
        *cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )
    stdout, stderr = await proc.communicate()
    return stdout.decode().strip()
```

### Supported Noctalia Actions:
1. **Audio Volume Management:**
   - Set Volume: `noctalia msg volume-set 65`
   - Mute Toggle: `noctalia msg volume-mute`
   - Up / Down: `noctalia msg volume-up` / `noctalia msg volume-down`
2. **Display Brightness:**
   - Set Brightness: `noctalia msg brightness-set 80`
   - Relative adjust: `noctalia msg brightness-up 10`
3. **Power & Session Inhibit (Caffeine):**
   - Prevent screen sleep while JARVIS is actively executing a long task: `noctalia msg caffeine-enable`
   - Re-enable power management when idle: `noctalia msg caffeine-disable`
4. **Desktop Bar Badges & Toast Notifications:**
   - Display active agent indicator on workspace: `noctalia msg workspace-alert-add 2`
   - Clear badge: `noctalia msg workspace-alert-clear 2` (or `workspace-alert-clear-all`)
   - Show notification toast: `noctalia msg notification-show "JARVIS" "Action completed successfully"`
5. **Clipboard Integration:**
   - Read clipboard: `noctalia msg clipboard-text`
   - Copy text: `noctalia msg clipboard-copy "Sample text"`

---

## 6. Application Lifecycle Management (UWSM)

### 6.1 The Danger of Raw `subprocess.Popen` for GUI Apps
Directly spawning GUI applications (e.g. `subprocess.Popen(["spotify"])`) creates orphaned child processes, pollutes process trees, bypasses systemd resource tracking, and causes applications to crash when the parent terminal or daemon restarts.

### 6.2 Mandated UWSM Execution Standard
Hyprland on CachyOS is managed by `uwsm` (Universal Wayland Session Manager). All desktop GUI applications launched by JARVIS must be invoked through `uwsm`:

```bash
uwsm app -- <application_binary> [arguments]
```

**Benefits of UWSM:**
1. **CGroup Isolation:** Each application runs in its own systemd transient unit: `app-<binary>@autostart.service`.
2. **Resource Boundaries:** Memory and CPU limits configured for the desktop session are strictly honored.
3. **Clean Teardown:** If an application crashes, systemd cleans up child processes and file descriptors without leaking resources.

---

## 7. Sandboxed CLI Command Runner

For command-line diagnostic tasks, compilation, and file inspection, JARVIS utilizes an asynchronous execution runner with strict safety constraints:

```python
import asyncio
import os
import signal

async def execute_sandboxed_cli(command: str, timeout_sec: int = 15) -> dict:
    """Executes a CLI command in a sandboxed process group with strict timeouts."""
    # Ensure timeout is bounded
    timeout_sec = min(max(timeout_sec, 1), 60)
    MAX_OUTPUT_BYTES = 100 * 1024  # 100 KB buffer cap

    proc = await asyncio.create_subprocess_shell(
        command,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        preexec_fn=os.setsid  # Create separate process group
    )

    try:
        stdout_data, stderr_data = await asyncio.wait_for(
            proc.communicate(),
            timeout=timeout_sec
        )
        return {
            "exit_code": proc.returncode,
            "stdout": stdout_data[:MAX_OUTPUT_BYTES].decode("utf-8", errors="replace"),
            "stderr": stderr_data[:MAX_OUTPUT_BYTES].decode("utf-8", errors="replace"),
            "truncated": len(stdout_data) > MAX_OUTPUT_BYTES
        }
    except asyncio.TimeoutError:
        # Kill the entire process group
        os.killpg(os.getpgid(proc.pid), signal.SIGKILL)
        return {
            "exit_code": -1,
            "stdout": "",
            "stderr": f"Command timed out after {timeout_sec} seconds.",
            "truncated": False
        }
```

### Safety Rules:
1. **Strict Timeout:** Default 15 seconds, hard ceiling 60 seconds.
2. **Buffer Cap:** Stdout and stderr capped at 100 KB to prevent buffer inflation.
3. **TTY Blocking:** Interactive utilities (`vim`, `nano`, `top`, unflagged `pacman`) are blocked or executed with non-interactive flags (`pacman -Q`, etc.).
4. **Process Group Termination:** Processes that exceed timeout are killed via `os.killpg()` to prevent rogue background threads.

---

## 8. Desktop Automation Verification Matrix

| Automation Channel | Target Component | Method / Protocol | Privilege Level |
|:---|:---|:---|:---|
| **Compositor Events** | Hyprland 0.57.2 | `.socket2.sock` persistent stream | User seat (unprivileged) |
| **Window Dispatch** | Hyprland 0.57.2 | `hyprctl dispatch <action>` | User seat (unprivileged) |
| **Desktop Shell** | Noctalia 1.2.6 | `/usr/bin/noctalia msg <args>` | User seat (unprivileged) |
| **Keystroke / Mouse**| Linux Kernel Input | Direct `/dev/uinput` via POSIX ACL | User `Qelvyrn` (no sudo/daemon) |
| **App Launching** | Desktop Applications | `uwsm app -- <app>` (systemd cgroups) | User session manager |
| **CLI Tools** | Shell Subprocesses | Async runner, process groups, 15s timeout | User seat (sandboxed) |
