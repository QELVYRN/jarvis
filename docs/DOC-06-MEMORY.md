# Document 06: Memory, Context & State Store Architecture (DOC-06-MEMORY)

**Document Reference:** `JARVIS-SPEC-DOC-06`  
**Classification:** Persistence, State & Memory Architecture  
**Target Environment:** SQLite 3.48+ WAL, In-Memory Ring Buffers, Gemini Embeddings  
**Status:** Phase 0 Verified — Approved Architecture Specification  

---

## 1. Executive Summary & Context Retention Goals

An agent operating without robust memory suffers from amnesia across restarts and lacks the ability to personalize interactions to the user's workflow. Conversely, unconstrained memory retention leads to context bloat, sluggish inference, ballooning API costs, and memory leaks on resource-constrained hardware.

Project JARVIS introduces a **tri-tier memory hierarchy** designed specifically for low RAM overhead (< 15 MB for memory subsystem) and high retrieval speed (< 5 ms SQLite index lookups).

---

## 2. Tri-Tier Memory Hierarchy

```
       +-------------------------------------------------------------------------+
       |                      TRI-TIER MEMORY ARCHITECTURE                       |
       +-------------------------------------------------------------------------+
       |                                                                         |
       |  +-------------------------------------------------------------------+  |
       |  | L1: WORKING CONTEXT (Python In-Memory RAM)                        |  |
       |  | - Focused window class, title & geometry (Hyprland socket2)       |  |
       |  | - Current active task objective & sub-step plan                   |  |
       |  | - Rolling 10-turn dialogue ring buffer                            |  |
       |  | - Footprint: < 15 MB RAM, 0 ms latency                            |  |
       |  +---------------------------------+---------------------------------+  |
       |                                    |                                    |
       |                     Periodic Flush & Session End                        |
       |                                    v                                    |
       |  +-------------------------------------------------------------------+  |
       |  | L2: EPISODIC & AUDIT STORE (SQLite 3.48+ WAL Mode)                |  |
       |  | - Full interaction logs, timestamps & token counts                |  |
       |  | - Action audit trail: tool calls, parameters, exit codes          |  |
       |  | - FTS5 full-text search index for fast text matching              |  |
       |  | - Path: ~/.local/share/jarvis/jarvis_memory.db                   |  |
       |  +---------------------------------+---------------------------------+  |
       |                                    |                                    |
       |                    Background Semantic Extraction                       |
       |                                    v                                    |
       |  +-------------------------------------------------------------------+  |
       |  | L3: LONG-TERM KNOWLEDGE STORE (Flat Vector Index / SQLite-vec)   |  |
       |  | - User preferences, system facts, common terminal aliases         |  |
       |  | - Embeddings via `gemini-embedding-001`                           |  |
       |  | - Flat cosine-similarity binary index (~2 MB disk)                |  |
       |  +-------------------------------------------------------------------+  |
       +-------------------------------------------------------------------------+
```

---

## 3. L2 Episodic & Audit Store: Complete SQLite Schema

The SQLite database resides at `/home/Qelvyrn/.local/share/jarvis/jarvis_memory.db`.
The database operates under Write-Ahead Logging (WAL) to permit concurrent reads during background writes without locking the main thread:
```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA busy_timeout = 5000;
PRAGMA temp_store = MEMORY;
```

### 3.1 Table Schemas

```sql
-- 1. Sessions Table
CREATE TABLE IF NOT EXISTS sessions (
    session_id TEXT PRIMARY KEY,
    start_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    end_time TIMESTAMP,
    primary_objective TEXT,
    summary_markdown TEXT,
    total_tokens_used INTEGER DEFAULT 0
);

-- 2. Interactions Table (Turns)
CREATE TABLE IF NOT EXISTS interactions (
    interaction_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    turn_index INTEGER NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    role TEXT CHECK(role IN ('user', 'assistant', 'system')),
    modality TEXT CHECK(modality IN ('voice', 'text', 'multimodal')),
    user_transcript TEXT,
    assistant_response TEXT,
    tokens_prompt INTEGER DEFAULT 0,
    tokens_completion INTEGER DEFAULT 0,
    FOREIGN KEY(session_id) REFERENCES sessions(session_id) ON DELETE CASCADE
);

-- 3. Action Audit Trail Table
CREATE TABLE IF NOT EXISTS action_audit (
    action_id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    tool_name TEXT NOT NULL,
    arguments_json TEXT NOT NULL,
    exit_code INTEGER,
    stdout_snippet TEXT,
    stderr_snippet TEXT,
    duration_ms INTEGER,
    hitl_approved BOOLEAN DEFAULT 1,
    FOREIGN KEY(session_id) REFERENCES sessions(session_id) ON DELETE CASCADE
);

-- 4. User Preferences & Desktop Bookmarks Table
CREATE TABLE IF NOT EXISTS desktop_bookmarks (
    bookmark_id INTEGER PRIMARY KEY AUTOINCREMENT,
    alias TEXT UNIQUE NOT NULL,
    file_path TEXT NOT NULL,
    description TEXT,
    usage_frequency INTEGER DEFAULT 1,
    last_accessed TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS user_preferences (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL,
    category TEXT DEFAULT 'general',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 5. Full-Text Search (FTS5) Virtual Table for Fast Context Retrieval
CREATE VIRTUAL TABLE IF NOT EXISTS interactions_fts USING fts5(
    user_transcript,
    assistant_response,
    content='interactions',
    content_rowid='rowid'
);
```

---

## 4. L3 Semantic Knowledge & Vector Retrieval

To prevent memory bloat, heavy external vector databases (such as Chroma, Milvus, or Qdrant) are **strictly forbidden**. On an 8GB machine with 2.6GB available RAM, running a Docker container or standalone JVM/Go vector server is unacceptable.

### 4.1 Lightweight Vector Architecture
- **Embedding Generation:** Uses cloud endpoint `gemini-embedding-001` (768-dimensional float32 vectors) or lightweight ONNX MiniLM-L6-v2 (< 60 MB RAM).
- **Index Storage:** Vectors are stored alongside metadata directly in SQLite or a flat binary file (`~/.local/share/jarvis/vectors.bin`).
- **Retrieval Engine:** Pure NumPy or SQLite dot-product calculation:
  ```python
  import numpy as np

  def search_similar_memories(query_vec: np.ndarray, top_k: int = 5):
      # Load memory matrix (e.g. 500 vectors x 768 dims = 1.5 MB RAM)
      similarities = np.dot(vectors, query_vec) / (np.linalg.norm(vectors, axis=1) * np.linalg.norm(query_vec))
      best_indices = np.argsort(similarities)[::-1][:top_k]
      return [metadata[i] for i in best_indices if similarities[i] > 0.65]
  ```

---

## 5. Context Pruning, Sliding Windows & Compaction

Even with Gemini's 1-million-token context window, packing hours of raw conversation into every prompt degrades latency, increases time-to-first-token (TTFT), and escalates API costs.

### 5.1 Sliding Window Compaction Policy
1. **Recent Turns (Turns 1–10):** Retained in pristine verbatim form in the working prompt context.
2. **Older Turns (Turns 11–30):** Summarized via an asynchronous `gemini-3.8-flash` condensation call:
   ```markdown
   ## Prior Conversation Summary:
   - User inquired about PipeWire buffer underruns in Discord.
   - JARVIS diagnosed sample rate mismatch and adjusted PipeWire default clock.
   - User opened VS Code at /home/Qelvyrn/Projects/test to test audio loop.
   ```
3. **Archived Turns (> 30 turns):** Flushed from prompt context entirely. Stored permanently in SQLite L2 store, queryable via FTS5 or L3 vector search on demand.

### 5.2 Header Pinning
The system instruction dynamically injects pinned contextual variables into the first prompt block:
- **Host Architecture:** Intel i3-1215U, CachyOS Kernel 7.3.4, Hyprland 0.57.2.
- **Active Focus:** `{ active_window_class }` - `{ active_window_title }`.
- **Active Audio Endpoint:** Sony WI-C100 (Bluetooth) vs. Alder Lake DMIC16kHz.
- **Workspace State:** Current workspace index and list of active window titles.

---

## 6. Memory Subsystem Verification Table

| Parameter / Feature | Design Specification | Performance Impact |
|:---|:---|:---|
| **L1 RAM Footprint** | < 15 MB RSS | Zero memory pressure on host. |
| **L2 SQLite Engine** | SQLite 3.48+ WAL mode | Non-blocking concurrent disk writes, < 2 ms commit. |
| **Search Mechanism** | SQLite FTS5 Full-Text Search | Sub-millisecond keyword lookup across all past logs. |
| **L3 Vector Store** | Flat NumPy / SQLite binary | ~1.5 MB RAM for 500 documents; no external server. |
| **Context Window** | Rolling 10-turn window + Markdown summary | Eliminates token bloat; stabilizes TTFT < 300 ms. |
| **Audit Compliance**| 100% of tool invocations logged | Complete traceability of every executed system command. |
