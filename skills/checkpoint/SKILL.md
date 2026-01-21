---
name: checkpoint
description: |
  Create checkpoint observations in claude-mem to save session context before /clear.
  Use when user wants to preserve current session state, create milestones, or save progress.
  Triggers: "save checkpoint", "create checkpoint", "checkpoint session", "save progress"
allowed-tools:
  - mcp__plugin_claude-mem_mem-search__help
  - mcp__plugin_claude-mem_mem-search__search
  - mcp__plugin_claude-mem_mem-search__get_observations
  - mcp__plugin_claude-mem_memory_create
  - mcp__plugin_claude-mem_memory_create_rich
user-invocable: true
---

# Checkpoint Skill for Claude Code

> **Version:** 2.12 (follows design.md version)
> **Design:** [../design.md](../design.md)
> **Session:** a77b77ae-ef2a-49f6-93d9-f78c8ac2d2f7

---

## Description

Create a checkpoint observation in claude-mem to save session context.

**Note:** This skill creates checkpoints only. User must manually run `/clear` after checkpointing if desired.

---

## Step 0: MCP Readiness Check (CRITICAL - DO NOT SKIP)

Before any checkpoint operation, verify claude-mem MCP is available and ready:

### Phase 0.1: Test MCP Connection
```
Call: mcp__plugin_claude-mem_mem-search__help
Expected: Returns help documentation

Call: mcp__plugin_claude-mem_mem-search__search
Params: { query: "test", limit: 1 }
Expected: Returns search results (may be empty)
```

### Phase 0.2: Test Database Accessibility
```
Call: mcp__plugin_claude-mem_memory_create
Params: {
  content: "Checkpoint connection test",
  metadata: { test: true },
  tags: ["test"]
}
Expected: Returns observation ID
```

### Phase 0.3: Validate Session Database
```
Check if database exists and is writable
→ MCP tool returns success = database OK
→ MCP error = database issue (see troubleshooting)
```

### If ANY check fails:
- Enter **Fallback Mode** (see Error Handling section)
- Use checkpoint_count = 0 (assume first checkpoint)
- Generate checkpoint ID with real timestamp
- Return checkpoint report in fallback format

### Success output:
```
MCP Status Check
═════════════════════════════════════
✅ Claude-Mem: Connected
✅ Database: Accessible
✅ Write Permission: Confirmed
═════════════════════════════════════
Proceeding with checkpoint creation...
```

---

## Implementation Flow

### Step 1: Session ID (Read from Environment)

**CRITICAL: Session ID อยู่ใน session context แล้ว - แค่อ่านใช้**

```
AI รู้ Session ID ของตัวเองอยู่แล้ว (อยู่ใน <env>)
แค่อ่านและใช้เลย:

session_id = "<อ่านจาก <env> Session ID>"
เช่น: a77b77ae-ef2a-49f6-93d9-f78c8ac2d2f7 (36 chars, UUID v4)
```

**ง่ายๆ เลย:**
- ✅ Session ID มีอยู่ใน environment แล้ว
- ✅ แค่อ่านและใช้
- ❌ ไม่ต้อง generate เอง
- ❌ ไม่ต้อง "ค้นหาวิธีอื่น"

**Test:** ลองถาม AI: "What is current session id?" → AI ตอบได้เลย

---

### Step 2: Checkpoint Count Detection (MCP Query-Based)

ดึงจำนวน checkpoints ที่มีอยู่ใน session ปัจจุบันจาก MCP:

```
Call: mcp__plugin_claude-mem_mem-search__search
Params: { query: "checkpoint session:{session_id}", limit: 100 }

checkpoint_count = len(results)
NEW_SEQ = checkpoint_count + 1
```

**เหตุผลของการใช้ MCP Query:**
- ✅ Stateless - ไม่ต้อง manage local state
- ✅ Accurate - นับจาก observations จริงใน MCP
- ✅ Survives restarts - state อยู่ใน MCP ไม่ใน local files

**Output:**
```
Checkpoint Count Detection
═════════════════════════════════════
📊 Found: N existing checkpoints
📝 Next sequence: {NEW_SEQ}
═════════════════════════════════════
```

---

### Step 3: Session History Retrieval (Hybrid Approach)

ใช้ **current session context** เป็นหลัก ถ้าไม่พอ → query MCP

```
Step 1: ตรวจสอบขนาด session ปัจจุบัน
current_context_size = estimate_context_size()

Step 2: เลือกวิธีการตามขนาด
if current_context_size < 50:  # Small session
    session_content = get_current_conversation_context()
elif current_context_size < 200:  # Medium session
    session_content = get_current_conversation_context()
    recent_observations = mcp__plugin_claude-mem_mem-search__get_recent_context(limit=20)
else:  # Large session
    session_content = mcp__plugin_claude-mem_mem-search__get_recent_context(limit=100)
```

---

### Step 4: AI-Powered Analysis (Template + Chain-of-Thought)

วิเคราะห์ session content ตาม structured template + CoT steps:

```
## Analysis Request Template

Step 1: Content Categorization
- จำแนกงาน: development/debugging/planning/documentation
- หา project domain: backend/frontend/devops/data

Step 2: Topic Extraction (Chain-of-Thought)
1. สแกน user messages → ดึง keywords, technical terms
2. สแกน assistant responses → ดึง technical concepts, actions
3. จับกลุ่ม keywords → สร้าง themes (3-7 topics)

Step 3: Outcome Identification
- Completed tasks: "Implemented X", "Fixed Y bug"
- Decisions made: "Chose A over B because..."
- Discoveries: "Found issue in Z", "Discovered workaround"
- Pending items: "TODO: test X", "REMINDER: review Y"

Step 4: Summary Generation
สรุปเป็น 3-5 ประโยค์:
1. งานหลักที่ทำคืออะไร?
2. สำเร็จได้อะไรบ้าง?
3. ปัจจุบัน state เป็นอย่างไร?

Step 5: Files and Tools
- Files referenced/modified: [list from context]
- Tools used: Read, Write, Bash, WebSearch, etc.
```

---

### Step 5: Generate Checkpoint ID

```
NEW_SEQ = checkpoint_count + 1
UNIX_MS = int(datetime.now(timezone.utc).timestamp() * 1000)

CHECKPOINT_ID = f"ccp-{session_id}-{NEW_SEQ:02d}-{UNIX_MS}"
```

**Format:** `ccp-<36-char-uuid>-<seq>-<13-digit-unix-ms>` (~56-58 chars)

---

### Step 6: MCP Tool Selection (Dynamic)

เลือก MCP tool ตาม use case:

```
Determine which MCP tool to use:

if needs_files or needs_rich_metadata:
    tool = mcp__plugin_claude-mem_memory_create_rich
    params = {
        "content": formatted_content,
        "metadata": checkpoint_metadata,
        "tags": tags,
        "attachments": file_attachments  # optional
    }
else:
    tool = mcp__plugin_claude-mem_memory_create
    params = {
        "content": formatted_content,
        "metadata": checkpoint_metadata,
        "tags": tags
    }

Call MCP tool
observation = tool(**params)
```

---

### Step 7: Create Observation (MCP Tool Call)

Format observation content ตาม template:

```markdown
Title: Checkpoint {seq}: {title} - Session {session_id}

Type: discovery

Content:
## Checkpoint Reference
- Checkpoint ID: {checkpoint_id}
- Session ID: {session_id} (UUID format)
- Sequence: {seq}
- Type: manual
- Created: {ISO_8601_TIMESTAMP}
- Created (Unix): {UNIX_TIMESTAMP_MS}

## Session Context
- Started: {session_start} (ISO)
- Started (Unix): {session_start_unix} (ms)
- Working Directory: {working_dir}
- Parent Session: {parent_session or none}

## Summary
{summary}

## Key Outcomes
- {outcome 1}
- {outcome 2}

## Files Referenced
- {file1}
- {file2}

## Tools Used
- {tool1}
- {tool2}

## Related Checkpoints
- Previous: (none)
- Next: (none)

## Retrieval Queries
# Restore this checkpoint
/mem-search query="checkpoint:{checkpoint_id}"

# All checkpoints in this session
/mem-search query="checkpoint session:{session_id}"

Metadata:
{
  "version": "2.3",
  "format": "claude-code-checkpoint",
  "checkpoint_id": "{checkpoint_id}",
  "session_id": "{session_id}",
  "sequence": {seq},
  "type": "checkpoint",
  "checkpoint_type": "manual",
  "created_at": "{ISO_TIMESTAMP}",
  "created_at_unix": {UNIX_MS},
  "parent_session": "{parent_session or null}",
  "topics": [{topics}],
  "files": [{files}],
  "tools_used": [{tools}],
  "working_directory": "{working_dir}",
  "hostname": "{hostname or null}"
}

Tags:
checkpoint, session-{session_id}, {topics}
```

---

### Step 8: Return Report

```
==============================================================
Checkpoint Created
==============================================================

Checkpoint ID: ccp-...-...-...
Session ID: {session_id}
Sequence: {seq}
Created: {ISO_8601_TIMESTAMP}

Observation ID: #{observation_id}

Restore this checkpoint:
  /mem-search query="checkpoint:{checkpoint_id}"

View all session checkpoints:
  /checkpoint --list

==============================================================
```

---

## List Mode Implementation

เมื่อ `--list`, `--parent`, หรือ `--all` options provided:

```
Determine query based on option:

if LIST_ALL:
    QUERY = f"checkpoint lineage:{root_session}"
elif LIST_PARENT and has_parent_session():
    parent_session = get_parent_session()
    QUERY = f"checkpoint (session:{session_id} OR session:{parent_session})"
else:
    QUERY = f"checkpoint session:{session_id}"

Call MCP search:
results = mcp__plugin_claude-mem_mem-search__search(
    query=QUERY,
    limit=100
)

Format and display results
```

**List Output Format:**

```
Checkpoints in this session:

  1. ccp-7670db3a-...-01-1737196625000
     Title: Initial Design
     Created: 2025-01-18 15:23:45
     Topics: checkpoint, design, session-management

  2. ccp-7670db3a-...-02-1737196960000
     Title: Implementation Started
     Created: 2025-01-18 15:16:00
     Topics: implementation, skills, mcp

Total: 2 checkpoints
```

---

## Restore Mode (--restore option)

เมื่อ user ใช้ `--restore`:

```
Step 1: Read current Session ID
session_id = "<อ่านจาก <env> Session ID>"

Step 2: Query MCP for latest checkpoint
results = mcp__plugin_claude-mem_mem-search__search(
    query=f"checkpoint session:{session_id}",
    limit=10
)

Step 3: Find latest checkpoint (sort by created_at_unix DESC)
if results:
    latest = sorted(results, key=lambda x: x.created_at_unix, reverse=True)[0]
    checkpoint_id = latest.metadata["checkpoint_id"]
else:
    print("ไม่พบ checkpoint สำหรับ session นี้")
    return

Step 4: Get checkpoint observation
observation = mcp__plugin_claude-mem_mem-search__get_observation(
    id=latest.observation_id
)

Step 5: Display checkpoint summary
print(f"Latest checkpoint: {checkpoint_id}")
print(f"Created: {observation.created_at}")
print(f"Summary: {observation.content[:200]}...")

Step 6: Suggest full restore
print(f"\nเพื่อ restore เต็ม:")
print(f"/mem-search query=\"checkpoint:{checkpoint_id}\"")
```

**Restore Output Format:**

```
==============================================================
Checkpoint Restored
==============================================================

Checkpoint ID: ccp-...-03-1737197220000
Created: 2025-01-18 15:27:00.000Z
Session: {session_id}

Summary:
{summary}

Key Outcomes:
- {outcome 1}
- {outcome 2}

Full restore:
  /mem-search query="checkpoint:{checkpoint_id}"

==============================================================
```

---

## Quick Reference

For complete MCP tool reference, error codes, and troubleshooting:
- See [reference.md](reference.md) for quick reference tables
- See [../design.md](../design.md) for complete design documentation

---

## Usage

```bash
/checkpoint [options]
```

### Options

| Option | Description |
|--------|-------------|
| `--title <title>` | Custom checkpoint title |
| `--tags <tags>` | Custom comma-separated tags |
| `--summary-only` | Only create master summary (no detailed observations) |
| `--include-files` | Include file change details |
| `--list` | List all checkpoints in current session (no creation) |
| `--parent` | Include checkpoints from parent session in list |
| `--all` | List all checkpoints in the entire lineage chain |
| `--restore` | Restore latest checkpoint from current session |

---

## Examples

```bash
# Create checkpoint with title
/checkpoint --title "API Design Complete"

# List all checkpoints
/checkpoint --list

# Restore latest checkpoint
/checkpoint --restore
```

---

## Error Handling

### MCP Connection Failed (Fallback Mode)

```
==============================================================
Checkpoint Created (Fallback Mode - MCP Unavailable)
==============================================================

⚠️  WARNING: MCP tools unavailable - using fallback mode

Checkpoint ID: ccp-{uuid}-01-{timestamp}
Session ID: {session_id}
Sequence: 1 (assumed - MCP unavailable)
Created: {timestamp}

Content:
## Summary
Checkpoint created in fallback mode due to MCP unavailability.
Session analyzed, checkpoint ID generated with real timestamp.
When MCP becomes available, run: /checkpoint --restore

## Key Outcomes
- Session analyzed in fallback mode
- Checkpoint ID: ccp-{uuid}-01-{timestamp}
- To migrate: /checkpoint --restore when MCP available

## Next Steps
1. Verify claude-mem MCP server is running
2. Check settings.json MCP configuration
3. Test with /checkpoint again

==============================================================
```

### No Observations Found

```
ℹ️ No checkpoints found for this session.

This is your first checkpoint. Proceeding with sequence 1.
```

### Checkpoint Creation Failed

```
❌ Error: Checkpoint creation failed

Details: {error message}

Troubleshooting:
1. Check claude-mem MCP server status
2. Verify MCP configuration in ~/.claude/settings.json
3. Ensure database exists at ~/.claude-mem/
4. Try: /mem-search --help (to test MCP connection)
```

---

## Environment Variables

| Variable | Description | Source |
|----------|-------------|--------|
| `CLAUDE_SESSION_ID` | Current session UUID (36 chars) | AI auto-detect from context |
| `CLAUDE_SESSION_START` | Session start time (ISO) | AI auto-detect from context |
| `PWD` | Working directory | System `os.getcwd()` |
| `HOSTNAME` | System hostname | System `os.environ.get("HOSTNAME")` |

---

## Key Design Decisions

### No SessionStart Hook Required
- ❌ ไม่ใช้ hooks
- ✅ AI auto-detects Session ID from context
- ✅ Simpler deployment - ไม่ต้อง setup hooks

### MCP-Based State Persistence
- ❌ ไม่มี local state files
- ✅ Query MCP หา checkpoint count
- ✅ State อยู่ใน MCP ทั้งหมด

### Hybrid Session History Retrieval
- ✅ ใช้ current context สำหรับ small sessions
- ✅ Query MCP สำหรับ large sessions
- ✅ Balance speed และ completeness

### AI-Powered Analysis
- ✅ Structured template + Chain-of-Thought
- ✅ ไม่ใช้ multi-agent (เพียงพอ)
- ✅ Consistent output format

### Dynamic MCP Tool Selection
- ✅ เลือก tool ตาม use case
- ✅ `memory_create` สำหรับ simple cases
- ✅ `memory_create_rich` สำหรับ complex cases

---

## Notes

- **Checkpoint ID Format**: `ccp-<uuid>-<seq>-<unix_ms>` (~56-58 characters)
- **Session ID**: From Claude Code CLI environment (<env> tags), UUID v4 (36 characters)
- **CRITICAL**: NEVER generate Session ID - it's provided by Claude Code CLI
- **Storage**: ALL state in claude-mem via MCP tools (no file storage, no hooks)
- **No scripts needed**: Markdown-based skill calls MCP tools directly
- **Built-in commands limitation**: `/clear` cannot be triggered by skills (it's a built-in command, not available through Skill tool)

---

> Full history: [../changelog.md](../changelog.md)
