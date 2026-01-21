# Claude Code Checkpoint Summary - Design Document

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Problem Statement](#problem-statement)
3. [Session ID System](#session-id-system)
4. [Checkpoint Reference System](#checkpoint-reference-system)
5. [Cross-Session Linking](#cross-session-linking)
6. [Retrieval Mechanism](#retrieval-mechanism)
7. [Metadata Schema](#metadata-schema)
8. [Data Collection & Analysis Strategy](#data-collection--analysis-strategy)
9. [Implementation Flow](#implementation-flow)
10. [API Specification](#api-specification)
11. [Examples](#examples)

---

## Architecture Overview

### Design Philosophy: **Simple & Leverage Existing**

Architecture Flow (Simple arrow style):

USER LAYER
├─ /checkpoint (slash command)
└─ /safe-clear (optional slash command)
  → Each command activates corresponding skill

SKILL IMPLEMENTATION LAYER
├─ SKILL.md (Markdown-based skill, Claude Code CLI spec)
└─ Responsibilities:
    ├─ Read session data (environment variables)
    ├─ Generate checkpoint ID
    ├─ Analyze session content (AI-powered)
    └─ Call MCP tools directly
  → Skills call MCP tools

MCP TOOLS LAYER (Existing - claude-mem)
├─ mcp__plugin_claude-mem_mcp-search__search
│  → Query observations
├─ mcp__plugin_claude-mem_mcp-search__get_observations
│  → Fetch detailed data
└─ Other claude-mem MCP tools
  → Tools already installed
  → Skills call these directly

STORAGE LAYER
└─ claude-mem (observations database)
   → Persistent storage
   → Searchable
   → Managed by existing MCP server

OPTIONAL HOOK LAYER
└─ guard_clear.py (minimal, optional)
   → Enforces /checkpoint before /clear
   → Also calls MCP tools
   → < 50 lines of code

Data Flow Summary:

User types /checkpoint
  → Skill activates
  → Reads environment variables
  → Generates checkpoint ID
  → Analyzes session content
  → Calls MCP tool directly
  → MCP stores in claude-mem
  → Returns result to user

### Key Principles

| ✅ ใช่ (ใช้สิ่งนี้) | ❌ ไม่ใช่ (หลีกเลี่ยง) |
|-------------------|----------------------|
| **Skills** เป็น entry point | ไม่ใช่ MCP server ใหม่ |
| **AI Auto-Detect** Session ID | ไม่ใช้ SessionStart hook |
| **MCP tools ที่มีอยู่แล้ว** | ไม่สร้าง Python scripts หนักๆ |
| **MCP Query-Based** state persistence | ไม่มี local state files |
| **Simple skill files** | ไม่ซับซ้อนด้วย infrastructure |
| **claude-mem เป็น storage** | ไม่ต้อง database ของตัวเอง |

### What Each Component Does

#### 1. Skills (User Interface Layer)

**Definition:** Slash commands ที่ user พิมพ์เพื่อใช้งาน

**Location:** `skills/` directory ใน Claude Code project

**Implementation (Markdown-Based Skill - Claude Code CLI Spec):**
- `SKILL.md` - Markdown-based skill definition (เราเลือกใช้)
  - Claude Code อ่านและ execute ตาม instructions ใน markdown โดยตรง
  - มี YAML frontmatter: `name`, `description`, `allowed-tools`, `user-invocable: true`
  - ไม่ต้องการ Python scripts เพิ่ม
  - เรียก MCP tools ผ่าน function calls โดย Claude

**Responsibilities:**
- **AI Auto-Detect Session ID** from current session context (no hooks required)
- Parse command options (`--title`, `--tags`, etc.)
- **Query MCP** หา checkpoint count (stateless approach)
- Generate checkpoint IDs
- Analyze session content (AI-powered analysis with CoT)
- **Select MCP tool dynamically** ตาม use case
- Call MCP tools directly (ไม่ใช่ scripts)
- Format and return results to user

#### 2. MCP Tools (Storage Layer - Existing)

**Definition:** MCP tools จาก `mcp__plugin_claude-mem` ที่ **ติดตั้งอยู่แล้ว**

**Available Tools:**
- `mcp__plugin_claude-mem_mcp-search__search` - ค้นหา observations
- `mcp__plugin_claude-mem_mcp-search__get_observations` - ดึงข้อมูลละเอียด
- (tools อื่นๆ ที่มีอยู่ใน claude-mem MCP)

**Usage:** Skills เรียก MCP tools เหล่านี้ **โดยตรง** ผ่าน function calls

**Note:** ไม่ต้อง:
- ❌ Start MCP server ของตัวเอง
- ❌ Config MCP connections
- ❌ Setup MCP infrastructure
- ✅ แค่เรียกใช้ tools ที่มีอยู่แล้ว

#### 3. Optional Hook (Minimal)

**Definition:** Lightweight script สำหรับ hook policies (ถ้าต้องการ)

**Example:** `guard_clear.py` - ป้องกัน `/clear` โดยไม่มี checkpoint

**Characteristics:**
- Minimal code (< 50 lines)
- ก็เรียก MCP tools เหมือนกัน
- Optional - ขึ้นกับว่าต้องการ enforcement หรือไม่

### Key Design Decisions

#### 1. Session ID from Environment (v2.8)

**Decision:** AI รู้ Session ID ของตัวเองอยู่แล้ว - อยู่ใน `<env>` tags

```text
✅ TRUE: Session ID อยู่ใน environment แล้ว
✅ TRUE: AI ตอบ "What is current session id?" ได้เลย
❌ FALSE: ต้อง generate เอง
❌ FALSE: ต้อง "ค้นหาวิธีอื่น"

Usage:
session_id = "<อ่านจาก <env> Session ID>"
```

**ความจริง:**
- AI Model รู้อยู่แล้วว่าทำงานอยู่ใน Claude Code CLI Session ID อะไร
- Session ID อยู่ใน `<env>` tags ที่ส่งมากับทุก request
- แค่อ่านและใช้ - ไม่ต้องยุ่งยาก

**Lesson Learned (v2.8):**
> อย่าทำให้เรื่องเกินไป - คำตอบอาจอยู่ตรงหน้า
>
> ❌ ผิด: ไป search วิธีอื่น ยุ่งกับ fallback mechanisms
> ✅ ถูก: AI รู้ Session ID ของตัวเองอยู่แล้ว แค่อ่านใช้
>
> **Test:** ถาม "What is current session id?" → AI ตอบได้ทันที

---

#### 2. MCP Query-Based State Persistence (v1.5.0)

**Decision:** Query MCP หา checkpoint count แทนที่ใช้ local state

```text
❌ NOT USED: Local file /tmp/claude-session-{session_id}.json
✅ USED: Query MCP observations หาจำนวน checkpoints

Flow:
Need checkpoint_count
  ↓
Query: mcp__plugin_claude-mem_mem-search__search(
         query="checkpoint session:{session_id}")
  ↓
Count results → checkpoint_count
```

**Benefits:**
- ✅ Stateless - ไม่ต้อง manage local files
- ✅ Accurate - นับจาก observations จริงใน MCP
- ✅ Survives restarts - state อยู่ใน MCP

---

#### 3. Hybrid Session History Retrieval (v1.5.0)

**Decision:** ใช้ current context สำหรับ small sessions, query MCP สำหรับ large sessions

```text
Session Size < 50 messages:
  → Use current context directly (fast)

Session Size 50-200 messages:
  → Hybrid: current context + MCP recent observations

Session Size > 200 messages:
  → Query MCP ดึง history ทั้งหมด
```

**Benefits:**
- ✅ Optimal performance - balance speed และ completeness
- ✅ Scalable - ทำงานได้กับทุกขนาด session
- ✅ No context loss - large sessions ไม่หายข้อมูล

---

#### 4. AI-Powered Analysis with CoT (v1.5.0)

**Decision:** Structured template + Chain-of-Thought steps (ไม่ใช้ multi-agent)

```text
Analysis Process:
1. Content Categorization (work type, domain)
2. Topic Extraction (keywords → themes)
3. Outcome Identification (completed, decisions, discoveries)
4. Summary Generation (3-5 sentences)
5. Files/Tools Tracking
```

**Benefits:**
- ✅ Consistent - template ensures standard output
- ✅ Traceable - CoT steps ชัดเจน
- ✅ Simple - ไม่ซับซ้อนด้วย multi-agent

---

#### 5. Dynamic MCP Tool Selection (v1.5.0)

**Decision:** เลือก tool ตาม use case (ไม่ใช้ใช้ tool เดียว)

```text
Simple checkpoint (no files):
  → mcp__plugin_claude-mem_memory_create

Rich checkpoint (with files/metadata):
  → mcp__plugin_claude-mem_memory_create_rich
```

**Benefits:**
- ✅ Flexible - ใช้ tool ที่เหมาะกับ situation
- ✅ Efficient - ไม่ over-engineering simple cases
- ✅ Future-proof - รองรับ files attachments

---

#### 6. Session Lineage & Post-Clear Restoration (v2.10)

**Decision:** `/clear` ไม่เปลี่ยน Session ID - ใช้ parent_session เชื่อมโยง

```text
Session A (original)
├─ Session ID: 7670db3a-2057-406a-a109-afcedef1cb97
├─ Checkpoint 1, 2, 3
└─ /clear

Session B (continuation)
├─ Session ID: 7670db3a-2057-406a-a109-afcedef1cb97 (SAME ID!)
├─ parent_session: null (root session)
└─ /checkpoint --restore → ค้นหาจาก Session A
```

**Key Points:**
- ✅ `/clear` ไม่เปลี่ยน Session ID
- ✅ Session B ใช้ ID เดียวกับ Session A
- ✅ Checkpoint metadata บันทึก parent_session เพื่อ trace lineage
- ✅ `/checkpoint --list` แสดง checkpoints ทั้งหมดของ session lineage

**Restore Commands:**
```bash
# ค้นหา checkpoint ล่าสุดทั้งหมด
/checkpoint --list --all

# Restore checkpoint ล่าสุด
/checkpoint --restore
```

**Benefits:**
- ✅ Session continuity - ID ไม่เปลี่ยน
- ✅ Easy restore - ค้นหาจาก checkpoint ID หรือ session ID
- ✅ Full lineage - trace ย้อนกลับได้ตลอด

---

### Data Flow Example: `/checkpoint` Command

```text
1. User types: /checkpoint --title "API Design Complete"
                 ↓
2. Skill activates (SKILL.md loaded by Claude Code)
                 ↓
3. AI Reads Session ID from Environment:
   - Reads <env> tags from session context
   - Session ID: 7670db3a-2057-406a-a109-afcedef1cb97 (provided by Claude Code CLI)
   - If missing → ERROR: Cannot proceed without Session ID
                 ↓
4. MCP Query for Checkpoint Count (Stateless):
   mcp__plugin_claude-mem_mem-search__search(
     query: "checkpoint session:7670db3a-2057-406a-a109-afcedef1cb97"
   )
   → Returns 2 existing checkpoints
   → NEW_SEQ = 3
                 ↓
5. AI Analyzes Session (Hybrid Approach):
   - Current context: ~30 messages → use directly
   - Extract topics: ["api", "design", "rest"]
   - Generate summary (CoT steps)
   - List files modified: [design.md, SKILL.md]
                 ↓
6. Generate Checkpoint ID:
   ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000
                 ↓
7. Dynamic MCP Tool Selection:
   - Simple checkpoint → mcp__plugin_claude-mem_memory_create
   - With metadata + tags
                 ↓
8. MCP Tool Call:
   mcp__plugin_claude-mem_memory_create(
     content: <formatted observation>,
     metadata: {...},
     tags: ["checkpoint", "session-7670db3a...", "api", "design"]
   )
                 ↓
9. MCP stores observation → Returns observation ID #8050
                 ↓
10. Skill Returns Report:
    ✅ Checkpoint created: ccp-...-03-...
    📝 Observation ID: #8050
    💾 State: Automatically tracked in MCP (no local state)
```

### Why This Architecture?

| Benefit | Explanation |
|---------|-------------|
| **Simple** | เขียน skill files เล็กๆ เรียก MCP tools ที่มีอยู่แล้ว |
| **No Infrastructure** | ไม่ต้อง setup/start/config MCP servers หรือ databases |
| **Leverages Existing** | ใช้ claude-mem + MCP tools ที่ติดตั้งอยู่แล้ว |
| **Maintainable** | Logic อยู่ใน skill files, storage อยู่ใน claude-mem |
| **Scalable** | claude-mem handle storage, เรา focus บน skill logic |

### What You DON'T Need

- ❌ **No MCP server** ของตัวเอง - ใช้ที่มีอยู่แล้ว
- ❌ **No Python scripts** ยาวๆ - skills เรียก MCP โดยตรง
- ❌ **No database setup** - claude-mem เป็น storage
- ❌ **No config files** เพียบ - minimal configuration
- ❌ **No complex infrastructure** - simple skills → MCP → storage

### Implementation Checklist

For Phase 2 (Implementation):

- [ ] Create `skills/checkpoint/SKILL.md` (following Claude Code CLI spec)
- [ ] (Optional) Create `.claude/hooks/guard_clear.py`
- [ ] Test MCP tool calls from skills
- [ ] Verify claude-mem storage works
- [ ] Update environment variables as needed

---

## Problem Statement

### Current Issue

Checkpoint ที่สร้างใน session หนึ่ง ไม่สามารถระบุได้แม่นยำว่า:
- เกิดจาก session ไหน (session ID อะไร)
- ถูกสร้างเมื่อไร (timestamp ไม่พอ)
- มี checkpoint กี่ตัว ใน session เดียวกัน
- checkpoint ไหนเป็น checkpoint สุดท้ายก่อน clear

### Impact

```
Session A (unknown ID)
├─ Checkpoint ??? (ไม่รู้ว่าของ session ไหน)
└─ ไม่สามารถ trace กลับได้แม่นยำ

Session B (new session)
├─ ค้นหา checkpoint เดิม
└─ ไม่พบ เพราะ reference ไม่ชัดเจน
```

### Requirements

- ✅ Session ID ที่ unique และ traceable
- ✅ Checkpoint ID ที่อ้างอิงถึง session ได้ชัดเจน
- ✅ สามารถระบุ checkpoint ล่าสุดของ session ได้
- ✅ สามารถ trace ประวัติ session ได้ (session lineage)

---

## Session ID System (Claude Code Native)

### Session ID Format

```
Format: <UUID v4>
Pattern: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
         8        4    4    4    12
Total: 36 characters (32 hex + 4 hyphens)

Example: 7670db3a-2057-406a-a109-afcedef1cb97

Entropy: 2^122 = 5.3×10^36 possibilities
Security: Excellent for session isolation

Source: Generated by Claude Code (UUID v4 standard)
```

### Session ID Retrieval

**รับจาก Environment Variable:**
```bash
# ใน hook หรือ skill สามารถอได้จาก:
${CLAUDE_SESSION_ID}
```

**หรือจาก Hook Context:**
```typescript
// SessionStart hook provides session_id
interface SessionStartEvent {
  session_id: string;  // UUID format
  source: string;       // "startup" | "clear"
  // ...
}
```

### Session Metadata v2.1

```typescript
interface SessionMetadata {
    session_id: string;        // UUID format (36 chars) - Claude Code native
    started_at: string;        // ISO 8601 timestamp
    started_at_unix: number;   // Unix timestamp (milliseconds) - NEW v2.1
    parent_session?: string;   // ถ้าเกิดจาก /clear อ้างถึง session เดิม
    checkpoint_count: number;  // จำนวน checkpoint ที่สร้างใน session นี้
    last_checkpoint?: string;  // Checkpoint ID ล่าสุด
    cleared: boolean;          // true ถ้า session ถูก clear แล้ว

    // NEW v2.1 fields
    working_directory: string;     // Current working directory
    hostname?: string;             // System hostname
    message_count: number;         // Total messages in session
    duration_seconds?: number;    // Session duration
}
```

### Terminology Note: “Session” มี 2 ชั้น (Claude Code vs claude-mem)

> จุดนี้ไม่ใช่ “ปัญหา” แต่เป็นการอธิบายศัพท์ให้ตรงกัน เพื่อไม่ให้ design ตีความผิดชั้น

- **Claude Code layer**: `/clear` จะ clear conversation history และ trigger `SessionStart` hooks ด้วย `source: "clear"` เพื่อให้ระบบ/ปลั๊กอิน re-inject context ได้
- **claude-mem layer**: รองรับ multi-prompt sessions; `/clear` ไม่ได้ end session ของ claude-mem แต่เป็นการ “ต่อ session เดิม” ด้วย `prompt_number` ใหม่ (prompt #1, #2, ...)

Design guidance:
- ให้ถือว่า `/clear` เป็น **boundary event** สำหรับ policy “checkpoint-before-clear”
- ถ้าจะเก็บ metadata เพิ่มเพื่อ debug/trace แนะนำบันทึกทั้ง:
  - `claude_code_session_id` (จาก hook input `session_id`)
  - `prompt_number` (ที่ claude-mem assign ต่อ prompt)

---

## Checkpoint Reference System

### Checkpoint ID Format

```text
Format: ccp-<session_uuid>-<seq>-<timestamp>

Components:
├─ Prefix: "ccp" (Claude Code Checkpoint)
├─ Session ID: UUID format (36 chars) - Claude Code native, full
├─ Sequence: ลำดับ checkpoint ใน session (01, 02, 03, ...)
└─ Timestamp: Unix timestamp (milliseconds)

Example: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
         └─ session: 7670db3a-2057-406a-a109-afcedef1cb97 (full UUID)
         └─ checkpoint ที่ 1
         └─ สร้าง 2025-01-18 15:23:45 (Unix ms)

Length: ccp- (4) + UUID (36) + -seq (3-4) + -timestamp (13-14)
       ≈ 56-58 characters total
```

### Checkpoint Metadata

```typescript
interface CheckpointMetadata {
    checkpoint_id: string;     // ccp-<uuid>-<seq>-<timestamp_ms>
    session_id: string;        // UUID format (36 chars, Claude Code native)
    sequence: number;          // 1, 2, 3, ...

    // Timestamps
    created_at: string;        // ISO 8601 timestamp
    created_at_unix: number;   // Unix timestamp (milliseconds)
    checkpoint_type: string;   // 'auto', 'manual', 'before-clear'

    // Content summary
    title: string;
    summary: string;           // สรุปเนื้อหา session
    topics: string[];
    key_outcomes: string[];    // ผลลัพธ์สำคัญ
    files_referenced: string[];

    // Observation references (ใน claude-mem)
    observation_ids: number[]; // [8044, 8045, 8046]

    // Session state at checkpoint time
    message_count: number;
    duration_seconds?: number;
    commands_executed?: number;
    tools_used?: string[];

    // Relations
    previous_checkpoint?: string; // checkpoint ID ก่อนหน้า
    next_checkpoint?: string;     // checkpoint ID ถัดไป (ถ้ามี)
    parent_session?: string;      // parent session ID (ถ้ามี)

    // System info (เพิ่ม v2.1)
    working_directory: string;
    hostname?: string;
    claude_code_version?: string;
}
```

### Checkpoint Types

| Type | Description | Trigger |
|------|-------------|---------|
| `auto` | Auto-checkpoint | Session ใหญ่เกินไป |
| `manual` | ผู้ใช้สร้างเอง | `/checkpoint` |
| `before-clear` | Before clear | Hook: `UserPromptSubmit` blocks `/clear` until `/checkpoint` exists |

---

## Cross-Session Linking

### Session Lineage

```text
Session A (7670db3a-2057-406a-a109-afcedef1cb97)
  ├─ Started: 2025-01-18T10:00:00Z
  ├─ Checkpoint 1: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196600000
  ├─ Checkpoint 2: ccp-7670db3a-2057-406a-a109-afcedef1cb97-02-1737196660000
  └─ /clear → เกิด Session B

Session B (b4e9c3d0-e5f6-a7b8-c9d0-e1f2a3b4c5d6)
  ├─ Started: 2025-01-18T10:20:30Z
  ├─ parent_session: 7670db3a-2057-406a-a109-afcedef1cb97
  ├─ Checkpoint 1: ccp-b4e9c3d0-e5f6-a7b8-c9d0-e1f2a3b4c5d6-01-1737197230000
  └─ อ้างถึง checkpoint สุดท้ายของ Session A
```

### Metadata Tracking

```typescript
interface SessionLineage {
    current_session: string;
    parent_session?: string;
    root_session?: string;        // session แรกของ lineage

    // ประวัติทั้งหมด
    session_chain: SessionLink[];
}

interface SessionLink {
    session_id: string;
    started_at: string;
    started_at_unix: number;
    checkpoint_count: number;
    last_checkpoint_id: string;
    duration_seconds: number;
}
```

---

## Retrieval Mechanism

### Search Patterns

```bash
# 1. หา checkpoint ล่าสุดของ session ปัจจุบัน
/mem-search query="checkpoint session:7670db3a-2057-406a-a109-afcedef1cb97 latest"

# 2. หา checkpoint ทั้งหมดของ session หนึ่ง
/mem-search query="checkpoint session:7670db3a-2057-406a-a109-afcedef1cb97"

# 3. หา checkpoint จาก session ก่อนหน้า
/mem-search query="checkpoint parent:7670db3a-2057-406a-a109-afcedef1cb97"

# 4. หา checkpoint ด้วย checkpoint ID
/mem-search query="checkpoint:ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000"

# 5. หา checkpoint ล่าสุดทั้งหมด
/mem-search query="checkpoint latest sort:desc"

# 6. หา checkpoint ด้วยช่วงเวลา
/mem-search query="checkpoint after:2025-01-18 before:2025-01-19"
```

### Observation Template สำหรับ Checkpoint

```text
Title: Checkpoint [seq]: <title> - Session <session_uuid>

Type: discovery

Content:
## Checkpoint Reference
- Checkpoint ID: ccp-<session_uuid>-<seq>-<timestamp_ms>
- Session ID: <session_uuid> (UUID format)
- Sequence: <seq> of <total>
- Type: <auto|manual|before-clear>
- Created: <ISO_8601_TIMESTAMP>
- Created (Unix): <UNIX_TIMESTAMP_MS>

## Session Context
- Started: <session_start_time> (ISO)
- Started (Unix): <session_start_unix_ms>
- Duration: <duration_seconds> seconds
- Messages: <count>
- Working Directory: <cwd>
- Parent Session: <parent_uuid or none>

## Summary
<session summary>

## Key Outcomes
- <outcome 1>
- <outcome 2>

## Files Referenced
- <file1>
- <file2>

## Tools Used
- <tool1>
- <tool2>

## Related Checkpoints
- Previous: <previous_checkpoint_id or none>
- Next: <next_checkpoint_id or none>

## Retrieval Queries
# Restore this checkpoint
/mem-search query="checkpoint:ccp-<session_uuid>-<seq>-<timestamp_ms>"

# All checkpoints in this session
/mem-search query="checkpoint session:<session_uuid>"

# Session lineage
/mem-search query="session lineage:<root_session_uuid>"

Metadata:
{
  "version": "2.1",
  "format": "claude-code-checkpoint",
  "checkpoint_id": "ccp-<session_uuid>-<seq>-<timestamp_ms>",
  "session_id": "<session_uuid>",
  "sequence": <seq>,
  "type": "checkpoint",
  "checkpoint_type": "<auto|manual|before-clear>",
  "created_at": "<ISO_TIMESTAMP>",
  "created_at_unix": <UNIX_MS>,
  "parent_session": "<parent_uuid or null>",
  "topics": ["<topic1>", "<topic2>"],
  "files": ["<file1>", "<file2>"],
  "tools_used": ["<tool1>", "<tool2>"],
  "observation_ids": [<id1>, <id2>],
  "latest_checkpoint": <true|false>,
  "working_directory": "<cwd>",
  "hostname": "<hostname or null>",
  "claude_code_version": "<version or null>"
}

Tags:
checkpoint, session-<session_uuid>, <auto-generated-topics>
```

---

## Metadata Schema

### Complete Checkpoint Record

```json
{
  "checkpoint": {
    "id": "ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000",
    "session_id": "7670db3a-2057-406a-a109-afcedef1cb97",
    "sequence": 1,
    "type": "manual",
    "created_at": "2025-01-18T15:23:45.000Z",
    "created_at_unix": 1737196625000,

    "content": {
      "title": "Session Checkpoint: Video Extension API Design",
      "summary": "ออกแบบระบบ checkpoint สำหรับ Claude Code...",
      "topics": ["checkpoint", "session-management", "claude-mem"],
      "key_outcomes": [
        "ระบุสาเหตุ session growth",
        "ออกแบบ checkpoint reference system"
      ],
      "files_referenced": ["design.video.md"]
    },

    "session_state": {
      "message_count": 12,
      "duration_seconds": 450,
      "files_modified": 1,
      "commands_executed": 3,
      "tools_used": ["Read", "Write", "Bash"]
    },

    "observations": {
      "master_summary_id": 8044,
      "detailed_ids": [8045, 8046],
      "total_count": 3
    },

    "relations": {
      "previous_checkpoint": null,
      "next_checkpoint": null,
      "parent_session": null,
      "root_session": "7670db3a-2057-406a-a109-afcedef1cb97"
    },

    "tags": ["checkpoint", "session-7670db3a-2057-406a-a109-afcedef1cb97", "claude-mem", "session-management"]
  }
}
```

### Session Record

```json
{
  "session": {
    "id": "7670db3a-2057-406a-a109-afcedef1cb97",
    "started_at": "2025-01-18T15:22:30.000Z",
    "started_at_unix": 1737196650000,
    "ended_at": null,
    "parent_session": null,
    "root_session": "7670db3a-2057-406a-a109-afcedef1cb97",

    "checkpoints": {
      "count": 1,
      "latest": "ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000",
      "all": ["ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000"]
    },

    "metadata": {
      "cleared": false,
      "auto_checkpoint_enabled": true,
      "working_directory": "/home/user/project",
      "hostname": "localhost"
    }
  }
}
```

---

## Data Collection & Analysis Strategy

### Overview

เมื่อสร้าง checkpoint ระบบต้องรวบรวมและวิเคราะห์ข้อมูลจาก session ปัจจุบันเพื่อสร้าง summary report ที่สามารถ restore context กลับมาได้อย่างยิ่งขึ้น

### Data Sources

```text
Session Data Sources:
├─ 1. Conversation History
│  ├─ User messages
│  ├─ Assistant responses
│  └─ System messages
│
├─ 2. File System Changes
│  ├─ Files created/modified/deleted
│  ├─ File paths
│  └─ File types
│
├─ 3. Tool Usage
│  ├─ Tools called (Read, Write, Bash, etc.)
│  ├─ Tool parameters
│  └─ Tool results
│
├─ 4. Environment Context
│  ├─ Working directory
│  ├─ Environment variables
│  ├─ Git status (if available)
│  └─ System state
│
└─ 5. Session Metadata
   ├─ Session ID (UUID)
   ├─ Timestamps
   ├─ Message count
   └─ Duration
```

### Collection Methods

#### 1. Conversation History Collection

**Access:**
```typescript
// จาก conversation history buffer
interface ConversationData {
    messages: Array<{
        role: 'user' | 'assistant' | 'system';
        content: string;
        timestamp: string;
        tool_calls?: ToolCall[];
    }>;
}
```

**Collection Strategy:**
- Buffer messages throughout session
- Include tool calls and results
- Capture user intent and assistant reasoning
- Store conversation tokens count

#### 2. File System Tracking

**Access:**
```bash
# Track file changes during session
git diff --name-only  # Modified files
git ls-files --others     # New files
git status                 # Full status
```

**Collection Strategy:**
- Monitor file operations (Read, Write)
- Track file paths accessed
- Note file types (code, config, data)
- Capture file modifications

#### 3. Tool Usage Monitoring

**Access:**
```typescript
interface ToolUsage {
    tool_name: string;
    count: number;
    parameters: Record<string, any>[];
    first_used: string;
    last_used: string;
}
```

**Tools to Track:**
- Read, Write, Edit - File operations
- Bash - Command execution
- Grep, Glob - Search operations
- WebSearch, WebFetch - External queries
- MCP tools - External service calls

### Analysis Pipeline

#### Stage 1: Content Categorization

```text
Input: Raw conversation + metadata

Process:
├─ 1. Identify main topics (NLP/keyword extraction)
│  └─ Output: ["session-growth", "api-design", "bugfix"]
├─ 2. Classify work type
│  └─ Output: "development" | "debugging" | "planning" | "documentation"
└─ 3. Detect project domain
│  └─ Output: "backend" | "frontend" | "devops" | "data"
```

#### Stage 2: Key Outcome Extraction

```text
Input: Categorized content

Process:
├─ 1. Identify completed tasks
│  └─ Extract: "Implemented X feature", "Fixed Y bug"
├─ 2. Identify decisions made
│  └─ Extract: "Chose API Z over W", "Decided on structure"
├─ 3. Identify important discoveries
│  └─ Extract: "Found issue in dependency", "Discovered workaround"
└─ 4. Identify pending items
│  └─ Extract: "TODO: test X", "REMINDER: review Y"
```

#### Stage 3: Summary Generation

```text
Input: Extracted outcomes

Process:
├─ 1. Generate executive summary (3-5 sentences)
│  ├─ What was the main goal?
│  ├─ What was accomplished?
│  └─ What's the current state?
├─ 2. Extract key technical details
│  ├─ Technologies used
│  ├─ Files modified
│  └─ Commands executed
└─ 3. Identify next steps
│  ├─ What to do next?
│  ├─ What's blocked?
│  └─ What decisions needed?
```

### Output Format Structure

#### Master Summary Observation

```text
Title: Checkpoint [seq]: <title> - Session <session_uuid>

Type: discovery

Content:
## Checkpoint Reference
- Checkpoint ID: ccp-<uuid>-<seq>-<unix_ms>
- Session ID: <session_uuid> (UUID format)
- Sequence: <seq> of <total>
- Type: <auto|manual|before-clear>
- Created: <ISO_8601_TIMESTAMP>
- Created (Unix): <UNIX_TIMESTAMP_MS>

## Session Context
- Started: <session_start_time> (ISO)
- Started (Unix): <session_start_unix_ms>
- Duration: <duration_seconds> seconds
- Messages: <count>
- Working Directory: <cwd>
- Parent Session: <parent_uuid or none>

## Summary
<session summary>

## Key Outcomes
- <outcome 1>
- <outcome 2>

## Files Referenced
- <file1>
- <file2>

## Tools Used
- <tool1>
- <tool2>

## Related Checkpoints
- Previous: <previous_checkpoint_id or none>
- Next: <next_checkpoint_id or none>

## Retrieval Queries
# Restore this checkpoint
/mem-search query="checkpoint:ccp-<uuid>-<seq>-<unix_ms>"

# All checkpoints in this session
/mem-search query="checkpoint session:<session_uuid>"

# Session lineage
/mem-search query="session lineage:<root_session_uuid>"

Metadata:
{
  "version": "2.1",
  "format": "claude-code-checkpoint",
  "checkpoint_id": "ccp-<uuid>-<seq>-<unix_ms>",
  "session_id": "<session_uuid>",
  "sequence": <seq>,
  "type": "checkpoint",
  "checkpoint_type": "<auto|manual|before-clear>",
  "created_at": "<ISO_TIMESTAMP>",
  "created_at_unix": <UNIX_MS>,
  "parent_session": "<parent_uuid or null>",
  "topics": ["<topic1>", "<topic2>"],
  "files": ["<file1>", "<file2>"],
  "tools_used": ["<tool1>", "<tool2>"],
  "observation_ids": [<id1>, <id2>],
  "latest_checkpoint": <true|false>,
  "working_directory": "<cwd>",
  "hostname": "<hostname or null>",
  "claude_code_version": "<version or null>"
}

Tags:
checkpoint, session-<session_uuid>, <auto-generated-topics>
```

#### Detailed Observations (Optional)

```text
Observation Type: analysis

Title: [Checkpoint <seq>] Detailed: <specific_topic>

Content:
## Topic: <topic_name>

### Details
<specific information about this topic>

### Code Examples
```language
<code snippets>
```

### Related Files
- <file1>: Description
- <file2>: Description

### Commands
- `<command>`: Description

Tags:
checkpoint-<checkpoint_id>, <topic_name>, <auto-tags>
```

### Analysis Process Flow

```text
1. Data Collection Phase
   ├─ Collect conversation history (full or partial)
   ├─ Collect file system changes
   ├─ Collect tool usage logs
   └─ Collect environment context

2. Analysis Phase (AI-Powered)
   ├─ Categorize content into topics
   ├─ Extract key outcomes
   ├─ Summarize main activities
   ├─ Identify files and tools used
   └─ Detect patterns and trends

3. Output Generation Phase
   ├─ Generate master summary observation
   ├─ Generate detailed observations (optional)
   ├─ Format metadata according to v2.1 schema
   └─ Store in claude-mem

4. Validation Phase
   ├─ Verify all required fields present
   ├─ Check UUID format validity
   ├─ Ensure timestamps are correct
   └─ Validate observation format
```

### Data Prioritization

**High Priority (Always Include):**
- Session metadata (ID, timestamps, duration)
- Main summary of what was done
- Key outcomes (3-7 items)
- Files modified/created
- Tools used

**Medium Priority (Include if relevant):**
- Detailed technical analysis
- Code examples
- Decision rationale
- Error messages and solutions

**Low Priority (Optional):**
- Minor conversational exchanges
- Failed attempts (unless informative)
- Redundant information

### Compression Strategy

**For Large Sessions (> 100 messages):**
1. Group related activities
2. Summarize by topic blocks
3. Extract only key code snippets
4. List files without full content
5. Focus on outcomes over process

**For Small Sessions (< 20 messages):**
1. Include more conversational context
2. Preserve detailed steps
3. Include more code examples
4. Full file references

---

## Implementation Flow

### Flow 1: เริ่ม Session ใหม่

```text
Session Start (Claude Code Native)
├─ Get Session ID from Claude Code
│  └─ <UUID v4> (36 characters)
├─ Create Session Record
│  ├─ session_id (UUID)
│  ├─ started_at (ISO)
│  ├─ started_at_unix (ms)
│  ├─ working_directory
│  ├─ parent_session (ถ้ามี)
│  └─ checkpoint_count = 0
└─ Store in accessible location
   └─ env var, temp file, หรือ memory
```

### Flow 2: สร้าง Checkpoint (MCP-Based)

```text
Checkpoint Creation (/checkpoint) - MCP-First Approach

Phase 1: Data Collection (from Environment & Context)
├─ Read current session metadata
│  ├─ session_id (UUID) from $CLAUDE_SESSION_ID
│  ├─ checkpoint_count from $CLAUDE_CHECKPOINT_COUNT (default: 0)
│  └─ working_directory from $(pwd)
├─ Collect conversation history (from session buffer)
│  ├─ User messages + Assistant responses
│  ├─ Tool calls and results
│  └─ Message timestamps
├─ Collect file system changes (git status)
│  ├─ Files modified (git diff)
│  ├─ Files created
│  └─ File paths accessed
├─ Collect tool usage (from session)
│  ├─ Tools called (Read, Write, Bash, etc.)
│  └─ Tool call counts
└─ Collect environment context
   ├─ Working directory
   ├─ Environment variables
   └─ System state

Phase 2: Analysis Pipeline (AI-Powered by Claude)
├─ Increment checkpoint_count
├─ Generate Checkpoint ID
│  └─ ccp-<session_uuid>-<seq>-<unix_timestamp_ms>
├─ Capture timestamps
│  ├─ created_at (ISO 8601)
│  └─ created_at_unix (ms)
└─ Analyze content (Claude AI)
   ├─ Stage 1: Content Categorization
   │  ├─ Extract topics
   │  ├─ Classify work type
   │  └─ Detect project domain
   ├─ Stage 2: Key Outcome Extraction
   │  ├─ Completed tasks
   │  ├─ Decisions made
   │  ├─ Discoveries found
   │  └─ Pending items
   └─ Stage 3: Summary Generation
      ├─ Executive summary
      ├─ Technical details
      └─ Next steps

Phase 3: MCP Storage (Direct MCP Calls - No Scripts)
├─ Format observation content
│  ├─ Checkpoint Reference section
│  ├─ Session Context section
│  ├─ Summary section
│  ├─ Key Outcomes section
│  ├─ Files Referenced section
│  ├─ Tools Used section
│  └─ Retrieval Queries section
├─ Call MCP: mcp__plugin_claude-mem_memory_create
│  ├─ content: <formatted observation>
│  ├─ metadata: {
│  │    version: "2.1",
│  │    format: "claude-code-checkpoint",
│  │    checkpoint_id: "ccp-<uuid>-<seq>-<unix_ms>",
│  │    session_id: "<session_uuid>",
│  │    sequence: <seq>,
│  │    type: "checkpoint",
│  │    checkpoint_type: "manual",
│  │    created_at: "<ISO_TIMESTAMP>",
│  │    created_at_unix: <UNIX_MS>,
│  │    parent_session: "<parent_uuid or null>",
│  │    topics: ["<topic1>", "<topic2>"],
│  │    files: ["<file1>", "<file2>"],
│  │    tools_used: ["<tool1>", "<tool2>"],
│  │    working_directory: "<cwd>",
│  │    hostname: "<hostname or null>"
│  │  }
│  └─ tags: ["checkpoint", "session-<uuid>", "<topics>"]
└─ Receive observation_id from MCP

Phase 4: Session State Update (Local Only)
├─ Update environment variables
│  ├─ export CLAUDE_CHECKPOINT_COUNT=$((checkpoint_count + 1))
│  └─ export CLAUDE_LAST_CHECKPOINT="<checkpoint_id>"
└─ Return checkpoint report to user

Output:
├─ Console checkpoint report
└─ Observation ID from MCP
```
│  └─ Per-topic deep dives
├─ Format metadata according to v2.1 schema
│  ├─ Include version: "2.1"
│  ├─ Include format: "claude-code-checkpoint"
│  └─ Include all required fields
└─ Store in claude-mem

Phase 4: Session State Update
├─ Update session record
│  ├─ checkpoint_count++
│  └─ last_checkpoint = new_checkpoint_id
└─ Return checkpoint report

Output:
├─ Console checkpoint report
└─ Observation IDs stored in claude-mem
```

**Link to:** Data Collection & Analysis Strategy section
- See "Data Collection & Analysis Strategy" above for detailed pipeline
```

### Flow 3: Session ใหม่หลัง Clear

**IMPORTANT:** `/clear` ไม่เปลี่ยน Session ID - ใช้ ID เดิมต่อ

```text
After /clear (same session continues)
├─ Session ID: SAME (ไม่เปลี่ยน!)
│  └─ เช่น: 7670db3a-2057-406a-a109-afcedef1cb97
├─ Conversation cleared แต่ Session ID ยังเดิม
├─ Checkpoint history ยังอยู่ใน claude-mem
└─ User สามารถ restore ด้วย:
    /checkpoint --list --all
    /checkpoint --restore
```

**Key Concept:**
- ❌ ผิด: /clear → สร้าง Session ID ใหม่
- ✅ ถูก: /clear → clear conversation เท่านั้น Session ID ยังเดิม

---

### Flow 4: Post-Clear Restoration

```text
User runs: /checkpoint --restore

Step 1: Read current Session ID
  session_id = "<อ่านจาก <env>>"
  → เช่น: 7670db3a-2057-406a-a109-afcedef1cb97

Step 2: Query MCP for latest checkpoint
  mcp__plugin_claude-mem_mem-search__search(
    query: "checkpoint session:{session_id}",
    limit: 10
  )
  → Returns checkpoints จาก session นี้ทั้งหมด

Step 3: Find latest checkpoint
  Sort by created_at_unix DESC
  → latest_checkpoint = ccp-7670db3a-...-03-1737197220000

Step 4: Retrieve and display
  mcp__plugin_claude-mem_mem-search__get_observations(
    ids: [latest_observation_id]
  )
  → Display checkpoint summary to user

Step 5: Suggest restore action
  "Found checkpoint from 2025-01-18 15:27"
  "Restore context? /mem-search query=\"checkpoint:{checkpoint_id}\""
```

**Alternative: List All Checkpoints**
```bash
/checkpoint --list --all
→ แสดง checkpoints ทั้งหมดของ session นี้เรียงตามเวลา
```

---

## API Specification

### Skill: /checkpoint (MCP-Based)

```bash
/checkpoint [options]

Options:
  --summary-only       Only create master summary
  --include-files      Include file change details
  --tags <tags>        Custom comma-separated tags
  --title <title>      Custom checkpoint title

Implementation (MCP-First):
├─ Get session metadata from environment variables
├─ Generate checkpoint ID: ccp-<session_uuid>-<seq>-<unix_ms>
├─ Call MCP: mcp__plugin_claude-mem_memory_create
│  ├─ content: Checkpoint observation (formatted)
│  ├─ metadata: { version: "2.1", checkpoint_id, session_id, ... }
│  └─ tags: ["checkpoint", "session-<uuid>", <topics>]
└─ Update environment: CLAUDE_CHECKPOINT_COUNT++, CLAUDE_LAST_CHECKPOINT

Note:
- No Python scripts required - uses MCP tools directly
- Observation stored in claude-mem via mcp__plugin_claude-mem_memory_create
- All state managed in MCP (no separate file storage)

Output:
  Console report + Observation ID from claude-mem
```

### Optional: Hook Policy for /clear Enforcement

> **Note:** This section describes OPTIONAL hook policy to encourage checkpointing before `/clear`. The `/checkpoint` skill itself does NOT trigger `/clear` (built-in commands cannot be automated).

**Purpose:** Encourage users to create checkpoint before clearing session

**Implementation (Optional - UserPromptSubmit Hook):**

```bash
# Hook checks for /clear command
# If no recent checkpoint exists: BLOCK with error message
# If checkpoint exists: Allow /clear to proceed
```

**User Override:**

```bash
/clear --force    # Bypass checkpoint requirement (use with caution)
```

**Use --force when:**
├─ /checkpoint failed due to technical issues (CKPT_001)
├─ claude-mem MCP server is down
├─ Network connectivity issues
└─ User explicitly wants to discard session without checkpoint

**Note:** This is OPTIONAL hook policy, NOT automation from `/checkpoint` skill

---

## Environment Variables

```bash
# Session tracking
CLAUDE_SESSION_ID="7670db3a-2057-406a-a109-afcedef1cb97"
CLAUDE_SESSION_START="2025-01-18T15:22:30.000Z"
CLAUDE_SESSION_START_UNIX="1737196650000"
CLAUDE_SESSION_PARENT="b4e9c3d0-e5f6-a7b8-c9d0-e1f2a3b4c5d6"
CLAUDE_SESSION_ROOT="7670db3a-2057-406a-a109-afcedef1cb97"
CLAUDE_SESSION_CWD="/home/user/project"

# Checkpoint tracking
CLAUDE_LAST_CHECKPOINT="ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000"
CLAUDE_CHECKPOINT_COUNT="1"
```

---

## Examples

### Example 1: Basic Checkpoint

```bash
# User runs
/checkpoint

# System output
Creating Session Checkpoint...

Session Information:
├─ Session ID: 7670db3a-2057-406a-a109-afcedef1cb97
├─ Started: 2025-01-18 15:22:30
├─ Started (Unix): 1737196650000
├─ Duration: 8 minutes (480 seconds)
├─ Messages: 15
├─ Working Directory: /home/user/project
└─ Checkpoints: 0 → 1

Analyzing session...
  Topics identified: session-growth, checkpoint-design
  Files referenced: design.md
  Tools used: Read, Write, Bash

Creating observations...
  Master summary: ID #8047
  Detailed: 2 observations

Checkpoint Created

Checkpoint ID: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
Session ID: 7670db3a-2057-406a-a109-afcedef1cb97
Sequence: 1 (first checkpoint)
Created: 2025-01-18 15:23:45.000Z

Observations:
  1. #8047 - Master Summary
  2. #8048 - Session Growth Analysis
  3. #8049 - Checkpoint Design

Restore this checkpoint:
  /mem-search query="checkpoint:ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000"

View all session checkpoints:
  /mem-search query="checkpoint session:7670db3a-2057-406a-a109-afcedef1cb97"

```

### Example 2: Safe Clear (Hook-enforced)

```bash
# User runs
/checkpoint --title "Ready for Testing"

# System output (summary)
- Checkpoint created: ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000
- Created: 2025-01-18 15:27:00.000Z
- Restore query:
  /mem-search query="checkpoint:ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000"

# User then runs
/clear

# Hook policy (UserPromptSubmit) behavior
- If user tries /clear without a recent checkpoint:
  → Hook blocks and asks user to run /checkpoint first
```

### Example 3: New Session After Clear

```bash
# New session starts

Session Information:
├─ Session ID: b4e9c3d0-e5f6-a7b8-c9d0-e1f2a3b4c5d6 (NEW)
├─ Started: 2025-01-18 15:30:45
├─ Parent: 7670db3a-2057-406a-a109-afcedef1cb97
├─ Root: 7670db3a-2057-406a-a109-afcedef1cb97
└─ Last Checkpoint: ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000

Quick restore available:
   /mem-search query="checkpoint:ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000"

Ready for new work.
```

### Example 4: Restore Context

```bash
# User in new session wants to restore
/mem-search query="checkpoint:ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000"

# System returns observation #8047

Checkpoint Restored

Checkpoint: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
From Session: 7670db3a-2057-406a-a109-afcedef1cb97
Created: 2025-01-18 15:23:45.000Z

Summary:
วิเคราะห์ปัญหา Claude Code session ที่เติบโตจนค้าง
และออกแบบ /checkpoint skill สำหรับสร้าง observation
checkpoint ก่อนการใช้ /clear

Key Outcomes:
- ระบุสาเหตุ: compact ไม่ลด file size
- ออกแบบ Session ID และ Checkpoint ID
- ออกแบบ retrieval mechanism

Related Observations:
  #8048 - Session Growth Root Cause
  #8049 - Checkpoint System Design

Continue work from this checkpoint?
```

### Example 5: Multiple Checkpoints in Session

```bash
# First checkpoint
/checkpoint --title "Initial Design"

Checkpoint: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196600000 (Sequence 1)
Created: 2025-01-18 15:10:00.000Z

# ... work continues ...

# Second checkpoint
/checkpoint --title "Implementation Started"

Checkpoint: ccp-7670db3a-2057-406a-a109-afcedef1cb97-02-1737196960000 (Sequence 2)
Created: 2025-01-18 15:16:00.000Z

# ... more work ...

# Third checkpoint
/checkpoint --title "Ready for Testing"

Checkpoint: ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000 (Sequence 3)
Created: 2025-01-18 15:27:00.000Z

# User clears session
/clear

# Hook policy blocks /clear unless a recent checkpoint exists

# In new session - find all
/mem-search query="checkpoint session:7670db3a-2057-406a-a109-afcedef1cb97"

Results:
  ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196600000 - Initial Design (15:10:00)
  ccp-7670db3a-2057-406a-a109-afcedef1cb97-02-1737196960000 - Implementation Started (15:16:00)
  ccp-7670db3a-2057-406a-a109-afcedef1cb97-03-1737197220000 - Ready for Testing (15:27:00) ← Latest
```

---

## Storage Strategy

### Where to Store Session Metadata

```text
Storage Options
├─ Environment Variables (fastest, session-scoped)
│  └─ $CLAUDE_SESSION_ID, $CLAUDE_LAST_CHECKPOINT
│
├─ Temporary Files (persistent across restarts)
│  └─ /tmp/claude-session-<pid>.json
│
├─ Claude-Mem Observations (permanent, searchable)
│  └─ Store session record as observation
│
└─ Combination (recommended)
   ├─ Environment for fast access
   ├─ Temp file for backup
   └─ Observation for long-term
```

### Recommended: Hybrid Storage

```javascript
// Pseudo-code
class SessionManager {
    constructor() {
        this.sessionId = this.getOrCreateSessionId();
        this.metadata = this.loadMetadata();
    }

    getOrCreateSessionId() {
        // 1. Check environment
        let id = process.env.CLAUDE_SESSION_ID;
        if (id) return id;

        // 2. Check temp file
        id = this.readTempFile();
        if (id) return id;

        // 3. Create new
        id = generateSessionId();
        this.saveToEnv(id);
        this.saveToTempFile(id);
        return id;
    }

    createCheckpoint(data) {
        const checkpoint = {
            id: generateCheckpointId(this.sessionId),
            session_id: this.sessionId,
            sequence: this.metadata.checkpoint_count + 1,
            ...data
        };

        // Save as observation
        this.createObservation(checkpoint);

        // Update metadata
        this.metadata.checkpoint_count++;
        this.metadata.last_checkpoint = checkpoint.id;
        this.saveMetadata();

        return checkpoint;
    }
}
```

---

## MCP Unavailable Fallback (v2.12)

### Scenario: MCP Server Not Running

```text
User runs: /checkpoint --title "Test"
  ↓
Skill checks MCP availability
  ↓
MCP tools unavailable → Use fallback mode
```

### Fallback Behavior

```text
Phase 1: MCP Check
└─ Try: mcp__plugin_claude-mem_mem-search__help
  ↓
  Error: MCP tools unavailable
  ↓
Phase 2: Fallback Mode
├─ Skip MCP query for checkpoint count
├─ Use checkpoint_count = 0 (assume first checkpoint)
├─ Generate checkpoint ID with real timestamp
├─ Analyze session content (AI-powered)
└─ Return mock observation format
```

### Fallback Output Format

```text
==============================================================
Checkpoint Created (Fallback Mode - MCP Unavailable)
==============================================================

⚠️  WARNING: MCP tools unavailable - using fallback mode

Checkpoint ID: ccp-<uuid>-01-<timestamp>
Session ID: <session_id>
Sequence: 1 (assumed - MCP unavailable)
Created: <timestamp>

Content:
## Summary
Checkpoint created in fallback mode due to MCP unavailability.
Session analyzed, checkpoint ID generated with real timestamp.
When MCP becomes available, run: /checkpoint --restore

## Key Outcomes
- Session analyzed in fallback mode
- Checkpoint ID: ccp-<uuid>-01-<timestamp>
- To migrate: /checkpoint --restore when MCP available

## Next Steps
1. Verify claude-mem MCP server is running
2. Check settings.json MCP configuration
3. Test with /checkpoint again

==============================================================
```

### Fallback Mode Limitations

| Feature | MCP Available | Fallback Mode |
|---------|----------------|---------------|
| Checkpoint count | ✅ Query from MCP | ⚠️ Assume 0 |
| Observation storage | ✅ Store in claude-mem | ❌ Mock output only |
| Checkpoint listing | ✅ Query from MCP | ❌ Not available |
| Checkpoint restore | ✅ Query from MCP | ❌ Not available |

### Recovery Procedure

```bash
# 1. Check MCP server status
ps aux | grep claude-mem

# 2. Check Claude Code settings
cat ~/.claude/settings.json | grep -A 10 "mcpServers"

# 3. Start/restart claude-mem if needed
claude-mem start

# 4. Test MCP connection
/checkpoint --title "MCP Test"

# 5. Migrate fallback checkpoints (if needed)
# Once MCP available, checkpoints will integrate automatically
```

---

## Edge Cases

### Case 1: Checkpoint แรกของ Session

```text
Session: 7670db3a-2057-406a-a109-afcedef1cb97
Checkpoint 1: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000

Previous checkpoint: (null) - ไม่มี
Sequence: 1 (first checkpoint)
```

### Case 2: Session ถูก clear กลางคัน

```text
Session A (incomplete)
├─ Checkpoint 1: ccp-<uuid>-01-...
├─ Checkpoint 2: ccp-<uuid>-02-...
└─ /clear (unexpected)

Session metadata จะบันทึกว่า cleared=true
Checkpoint ที่สร้างไปยังคงอยู่ใน claude-mem
```

### Case 3: Multiple /clear ติดกัน

```text
Session A → /clear → Session B → /clear → Session C

Lineage:
Session C
├─ parent: Session B
└─ root: Session A

Can trace back:
/mem-search query="session lineage:<uuid_a>"
```

### Case 4: Manual Session ID (debugging)

```bash
# Override session ID for testing (must use UUID format)
export CLAUDE_SESSION_ID="00000000-0000-0000-0000-000000000001"

/checkpoint

Checkpoint ID: ccp-00000000-0000-0000-0000-000000000001-01-...
Session ID: 00000000-0000-0000-0000-000000000001
```

### Case 5: Deadlock Scenario - Checkpoint Failure

```text
DEADLOCK RISK:
User types: /clear
  → Hook: Has checkpoint? NO → BLOCK
User tries: /checkpoint
  → FAILS! (claude-mem error, network down, etc.)
User tries: /clear again
  → Hook blocks again (no checkpoint)
User is TRAPPED - DEADLOCK

SOLUTION: /clear --force flag

/clear --force
  → Hook: Force flag detected → BYPASS checkpoint check
  → Allow /clear immediately
  → Warning: "Proceeding without checkpoint"

Implementation:
├─ Hook checks for --force flag first
├─ If --force present: allow immediately
├─ If --force absent: normal checkpoint check
└─ User can escape deadlock with explicit intent
```

**When to use `/clear --force`:**
- `/checkpoint` fails due to technical issues (CKPT_001)
- claude-mem MCP server is down
- Network connectivity issues
- User explicitly wants to discard session without checkpoint

**Security Consideration:**
- `--force` flag requires explicit user intent
- Warning message ensures user understands risk
- Audit log records force clear events
- Can be disabled in production if needed

---

## Best Practices

### When to Create Checkpoints

```text
Checkpoint Triggers
├─ Manual (/checkpoint)
│  └─ ผู้ใช้ตัดสินใจเอง
│
├─ Before Clear (Hook-enforced)
│  └─ Hook blocks /clear unless a recent /checkpoint exists
│
└─ Auto (future feature)
   └─ Session size > threshold
```

### Checkpoint Frequency

```text
Guidelines:
├─ Small session (< 50 messages)
│  └─ Checkpoint เมื่อจะ clear หรือจบงาน
│
├─ Medium session (50-200 messages)
│  └─ Checkpoint ทุก milestone สำคัญ
│
└─ Large session (> 200 messages)
   └─ Checkpoint ทุก 30-50 messages
```

### Tagging Strategy

```text
Checkpoint Tags
├─ checkpoint (always)
├─ session-<YYYYMMDD> (always)
├─ <auto-topics> (from content)
└─ <custom-tags> (if user provides)

Examples:
checkpoint, session-20250118, video-api, gemini, design
checkpoint, session-20250118, bugfix, authentication
checkpoint, session-20250118, refactor, performance
```

---

## Future Enhancements

### Phase 2 Features

```text
Planned Features
├─ Auto-checkpoint
│  └─ Auto checkpoint เมื่อ session ใหญ่
│
├─ Checkpoint diff
│  └─ เปรียบเทียบ checkpoint 2 ตัว
│
├─ Checkpoint merge
│  └─ รวม checkpoint หลายตัวเป็นหนึ่ง
│
├─ Visual session timeline
│  └─ แสดง lineage แบบกราฟิก
│
└─ Cross-session search
   └─ ค้นหาทั้ง lineage พร้อมกัน
```

### Phase 3 Features

```text
Advanced Features
├─ Checkpoint branching
│  └─ เปรียบเทียบ alternate paths
│
├─ Checkpoint sharing
│  └─ Export checkpoint ให้ session อื่น
│
├─ Smart suggestions
│  └─ แนะนำ checkpoint ที่เกี่ยวข้อง
│
└─ Integration with Memora
   └─ Sync checkpoint เป็น knowledge graph
```

---

## References

### Related Documents

- [skills/checkpoint/reference.md](skills/checkpoint/reference.md) - Quick reference tables for MCP tools, error codes, and troubleshooting
- [claude-mem Documentation](https://github.com/darkwingtm/claude-mem)
- [Memora Memory Graph](https://github.com/memora-ai/memora)
- [Claude Code Session Management](../)

### Related Skills

- `/checkpoint` - Create checkpoint or list checkpoints
- `/mem-search` - Search and retrieve observations
- `/sync-memora` - Sync to long-term storage

---

> Full history: [changelog.md](./changelog.md)

---

## Appendix A: ID Formats Reference

### Quick Reference

```text
Session ID (Claude Code Native - UUID):
  Format: <UUID v4>
  Pattern: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Length: 36 characters (32 hex + 4 hyphens)
  Example: 7670db3a-2057-406a-a109-afcedef1cb97
  Source: Claude Code native (UUID v4 standard)
  Entropy: 2^122 = 5.3×10^36 possibilities

Checkpoint ID v2.1:
  Format: ccp-<session_uuid>-<seq>-<unix_ms>
  Length: ~56-58 characters
  Example: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
  Parts:
    - ccp: Checkpoint prefix (4 chars)
    - session_uuid: Full UUID (36 chars)
    - seq: Sequence number with prefix (3-4 chars)
    - unix_ms: Unix timestamp in milliseconds (13 chars)

Query Formats:
- checkpoint:ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
- checkpoint session:7670db3a-2057-406a-a109-afcedef1cb97
- checkpoint latest
- session lineage:7670db3a-2057-406a-a109-afcedef1cb97
```

---

## Appendix B: Error Codes

| Code | Description | Solution |
|------|-------------|----------|
| `SESS_001` | Cannot generate session ID | Retry or specify manually |
| `SESS_002` | Session metadata corrupted | Reinitialize session |
| `CKPT_001` | Checkpoint creation failed | Check claude-mem status |
| `CKPT_002` | Checkpoint ID collision | Regenerate with new timestamp |
| `META_001` | Cannot save session metadata | Check disk permissions |

---

## Appendix C: Built-in Commands Limitation

### Why `/clear` Cannot Be Automated

**Finding Date:** 2026-01-21
**Source:** [Claude Code Slash Commands Documentation](https://code.claude.com/docs/en/slash-commands)
**Research Method:** Jina MCP deep search + claude-mem observation analysis

### Key Facts:

1. **Built-in commands NOT available through Skill tool**
   - Commands like `/clear`, `/compact`, `/init`, `/help` are built-in
   - These live in "interactive mode", NOT in the skills system
   - Custom slash commands CAN be invoked programmatically (via Skill tool)
   - Built-in commands CANNOT be invoked programmatically

2. **Official Documentation Quote:**
   > "Built-in commands like `/compact` and `/init` are **not available through the Skill tool**"

3. **Research Confirmation:**
   - claude-mem observation #8082: "Confirmed Inability to Automate `/clear` via Skills or Hooks"
   - claude-mem observation #8084: "Re-confirmed `/clear` cannot be automated via Hooks or Skills"
   - claude-mem observation #8085: "Detailed Analysis Confirms Inability to Automate `/clear` via Hooks"

### Design Implications:

✅ **CORRECT:** `/checkpoint` creates checkpoints ONLY - user manually runs `/clear`
❌ **NOT POSSIBLE:** `/checkpoint --clear` to automate session clearing
✅ **ALTERNATIVE:** Hook policy to BLOCK `/clear` until checkpoint exists (user manually runs `/clear` after checkpointing)

### Lesson Learned:

> **Critical Feasibility Check:** Always verify API capabilities before designing features
>
> **Research Approach:** Use multiple sources (official docs + memory search + web search)
>
> **Design Impact:** When in doubt, simplify - manual user action is sometimes the best solution

---

**End of Design Document v2.12**

---

> Full history: [changelog.md](changelog.md)
