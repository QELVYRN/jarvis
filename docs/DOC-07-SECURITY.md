# Document 07: Security, Privacy & Permission Boundary Model (DOC-07-SECURITY)

**Document Reference:** `JARVIS-SPEC-DOC-07`  
**Classification:** Security Architecture & Safety Governance  
**Target Environment:** CachyOS Linux, Wayland, Gemini API Endpoints  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Security Philosophy

An agent with voice interaction, screen perception, and desktop tool execution powers presents severe security and privacy vectors if left unconstrained:
- **Prompt Injection:** An adversary could display malicious text on a web page or in a terminal output designed to instruct the model to execute destructive bash commands (e.g., *"Delete all files in home"*).
- **Accidental Destruction:** LLM hallucination or ambiguous user commands could trigger accidental data loss (`rm`, formatting drives).
- **Data Exfiltration:** Sensitive credentials, API keys, or private communications could be transmitted across the cloud multimodal boundary.

Project JARVIS adheres to the principle of **Defense-in-Depth** and **Zero-Trust Autonomous Execution for Destructive Operations**.

---

## 2. Autonomous vs. Human-in-the-Loop (HITL) Execution Matrix

All operations executable by JARVIS are strictly categorized into three deterministic risk tiers:

```
       +-------------------------------------------------------------------------+
       |                        SECURITY RISK TIER MATRIX                        |
       +--------+----------------------------+-----------------------------------+
       | TIER 0 | READ-ONLY / OBSERVATION    | Screen capture, window query,     |
       |        | (Fully Autonomous)         | reading files, volume check       |
       +--------+----------------------------+-----------------------------------+
       | TIER 1 | BENIGN DESKTOP MUTATIONS   | Workspace switch, app launch      |
       |        | (Autonomous + Visual Toast)| (UWSM), volume/brightness set     |
       +--------+----------------------------+-----------------------------------+
       | TIER 2 | DESTRUCTIVE / HIGH RISK    | rm -rf, git push -f, sudo,        |
       |        | (Strict HITL Confirmation) | systemctl stop, modifying configs |
       +--------+----------------------------+-----------------------------------+
```

### 2.1 Tier Classification Breakdown

| Tier | Classification | Permitted Actions | Guardrail Mechanism |
|:---|:---|:---|:---|
| **Tier 0** | **Read-Only / Inspection** | Screen snapshot via `grim`, active window metadata query via `hyprctl`, file reading, process listing (`ps`), reading audio volume, web search. | **Fully Autonomous:** Executes immediately; results returned to cognitive pipeline without prompting. |
| **Tier 1** | **Benign Desktop Operations** | Changing workspace, focusing window, launching whitelisted apps (`uwsm app -- spotify`), setting volume/brightness via `noctalia msg`, copying to clipboard. | **Autonomous with Notification:** Executes immediately; triggers an unobtrusive Noctalia toast notification or top-bar badge indicating the executed action. |
| **Tier 2** | **Destructive / High-Privilege Actions** | File deletion (`rm`, `unlink`), overwriting source files, `git push --force`, system shutdown/reboot, running any command requiring `sudo`, editing system configs (`/etc`), installing packages. | **Strict HITL Gate:** Suspends execution immediately. Emits visual confirmation dialog and vocal prompt: *"JARVIS requests confirmation to run: `<command>`. Proceed? (Yes/No)"*. Action remains blocked until explicit user confirmation. |

---

## 3. Human-in-the-Loop (HITL) Enforcement Protocol

When a tool classified under Tier 2 is invoked by either `gemini-3.8-flash` or `gemini-3.8-live`:

1. **Pipeline Pause:** Tool execution halts; the action is assigned a temporary `action_uuid`.
2. **Visual Desktop Modal:** JARVIS invokes Noctalia's dialog interface or a lightweight GTK dialog (`zenity` / Noctalia modal) displaying:
   - Tool Name and Target Target Path / Command.
   - Proposed Arguments.
   - Exact Bash Command string.
   - Danger Rating: `[WARNING: DESTRUCTIVE OPERATION]`.
3. **Vocal Prompt:** The reactive audio engine outputs:
   > *"I have prepared the command to delete the build directory. Please confirm on screen or say 'confirm' to execute."*
4. **Resolution:**
   - If the user clicks **Confirm** or speaks **"Confirm" / "Proceed"**: The command runs, and the result is logged to SQLite with `hitl_approved = 1`.
   - If the user clicks **Cancel** or 30 seconds elapse without response: The action is **aborted**, returning `{ "status": "aborted_by_user" }` to the model.

---

## 4. Credential Protection & Secret Redaction

### 4.1 Headless Key Sourcing
- The Gemini API key must never be hardcoded or passed via CLI flags visible in `ps` process tables.
- **Key Path:** `/home/Qelvyrn/.config/jarvis/env`
- **Permissions:** `0600` (`-rw-------`, readable only by user `Qelvyrn`).
- **Systemd Integration:** When running as a user service, keys are loaded via:
  ```ini
  EnvironmentFile=%h/.config/jarvis/env
  ```
- This completely avoids GUI D-Bus Secret Service unlocking prompts during headless startup or system boot.

### 4.2 Real-Time Output & Ingestion Sanitization
All text emitted by subshells, file reads, or error logs passes through a compiled regex sanitization filter before being sent over the WebSocket to Gemini or written to SQLite episodic memory:

```python
import re

SECRET_PATTERNS = [
    re.compile(r"AIza[0-9A-Za-z-_]{35}"),               # Google Gemini / Cloud API Keys
    re.compile(r"ghp_[0-9a-zA-Z]{36}"),                 # GitHub Personal Access Tokens
    re.compile(r"-----BEGIN [A-Z]+ PRIVATE KEY-----"),   # SSH / RSA Private Keys
    re.compile(r"password\s*[:=]\s*['\"]?[^\s'\"]+", re.I),
    re.compile(r"bearer\s+[a-zA-Z0-9_\-\.]+", re.I),
]

def sanitize_text(text: str) -> str:
    redacted = text
    for pattern in SECRET_PATTERNS:
        redacted = pattern.sub("[REDACTED_SECRET]", redacted)
    return redacted
```

---

## 5. Wayland Security Compliance

Modern Wayland deliberately eliminates the X11 architectural security holes where any client could snoop keystrokes from other windows or inject rogue events into banking software:

1. **No Global Keylogging:** JARVIS does not attempt to hook global keyboard input streams. Push-to-Talk is handled through compositor-level keybindings in `~/.config/hypr/config/binds.lua` (`hl.bind(mainMod .. " + G", ...)`), triggering clean IPC signals to `jarvis.sock`.
2. **Authorized Screen Capture:** Screen capture requests route through compositor-approved protocols (`grim` with `wlr-screencopy-unstable-v1` or `xdg-desktop-portal-hyprland`), ensuring compositor authority is maintained.
3. **Restricted Virtual Input:** Synthetic typing via `/dev/uinput` is strictly bounded to the user's active seat, governed by Linux kernel POSIX ACLs.

---

## 6. Security Architecture Verification Table

| Security Vector | Risk Rating | Architectural Defense |
|:---|:---|:---|
| **Prompt Injection via Screen/Text** | High | Strict Tier 2 HITL confirmation gate on all mutations. |
| **Accidental System Destruction** | High | Destruction whitelist (`rm`, `dd`, `mkfs` require modal click). |
| **API Key Leakage** | High | File permissions `0600`, regex secret redactor on all I/O. |
| **Visual Privacy Leakage** | Medium | Window class blacklist (password managers/private browser masked). |
| **Rogue Child Processes** | Medium | Process groups (`os.setsid`), 15s timeout, UWSM systemd units. |
| **X11 Snooping / Hijacking** | Low | Native Wayland protocol isolation; zero XWayland security bypasses. |
