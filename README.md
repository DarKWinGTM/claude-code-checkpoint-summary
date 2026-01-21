# Claude Code Checkpoint Summary System

## Overview

ระบบ checkpoint สำหรับ Claude Code sessions ที่ช่วยให้:
- Track session history ได้อย่างยิ่งขึ้นด้วย UUID-based Session IDs
- Save checkpoints เป็น observations ใน claude-mem
- Restore context จาก sessions ก่อนหน้าได้อย่างยิ่งขึ้น
- Trace session lineage ผ่าน `/clear` operations

## Version

**Current Version: v2.12** (2026-01-21)

> Full history: [changelog.md](./changelog.md)

> Full history: [changelog.md](./changelog.md)

## Problem Statement

### Issues ที่ระบบแก้ได

1. **Session ID ไม่ unique พอ** - ไม่สามารถระบุได้ว่า checkpoint มาจาก session ไหน
2. **ไม่มี timestamp ที่ชัดเจน** - ยากต่อการ trace ประวัติ
3. **Session lineage ไม่ชัดเจน** - ไม่รู้ว่า session ไหนเกิดจาก session ไหน
4. **Checkpoint ล่าสุดไม่ชัดเจน** - ไม่รู้ว่า checkpoint ไหนคือล่าสุดก่อน clear

### Solution

```
Session ID (UUID v4) → Checkpoint ID (ccp-<uuid>-<seq>-<unix_ms>)
       ↓                         ↓
Traceable          Restoreable
```

## Key Features

### 1. UUID-based Session Identification

```
Session ID: 7670db3a-2057-406a-a109-afcedef1cb97 (36 chars)
Entropy: 2^122 = 5.3×10^36 possibilities
Source: Claude Code native (UUID v4 standard)
```

### 2. Checkpoint Reference System

```
Checkpoint ID: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
Components:
├─ ccp: Checkpoint prefix
├─ session_uuid: Full UUID (36 chars)
├─ seq: Sequence number (01, 02, 03, ...)
└─ unix_ms: Unix timestamp in milliseconds (13 chars)
```

### 3. Session Lineage Tracking

```
Session A (root)
├─ Checkpoint 1, 2
└─ /clear
   ↓
Session B (child)
├─ parent_session: Session A UUID
└─ Can trace back to root
```

### 4. Cross-Session Restoration

```bash
# Restore specific checkpoint
/mem-search query="checkpoint:<checkpoint_id>"

# Find all checkpoints in session
/mem-search query="checkpoint session:<session_uuid>"

# Trace session lineage
/mem-search query="session lineage:<root_uuid>"
```

## Usage

### Basic Checkpoint

```bash
/checkpoint
```

### Checkpoint with Custom Title

```bash
/checkpoint --title "Feature Implementation Complete"
```

### View Session Info

```bash
/session-info
```

### Restore Context

```bash
# In new session, run:
/mem-search query="checkpoint:<checkpoint_id>"

# System returns:
# - Session summary
# - Key outcomes
# - Related observations
# - Files referenced
```

## Architecture

### Components

1. **Session Manager** - Track session state (UUID, start time, checkpoints)
2. **Checkpoint Generator** - Create checkpoint IDs and observations
3. **Storage Layer** - claude-mem observations
4. **Retrieval Mechanism** - /mem-search queries

### Data Flow

```
Session Start
  ↓
Generate UUID (Claude Code native)
  ↓
User works...
  ↓
User runs /checkpoint
  ↓
Create observation in claude-mem
  ↓
User runs /clear
  ↓
New session (new UUID)
  ↓
Can restore from checkpoint
```

## Design Document

See [design.md](./design.md) for:
- Complete technical specifications
- Metadata schemas
- Implementation flows
- API specifications
- Examples
- Edge cases

## Status

**v2.12** - Production Ready

- [x] UUID-based Session IDs
- [x] Checkpoint Reference System
- [x] Session Lineage Tracking
- [x] Cross-Session Restoration
- [x] Observation Templates
- [x] MCP Readiness Check & Fallback Mode
- [x] Quick Reference Documentation (reference.md)
- [x] Rules-Compliant Documentation Structure

## Dependencies

- **Claude Code** - Native UUID generation
- **claude-mem** - Observation storage and retrieval
- **MCP Servers** - For integration hooks

## Future Enhancements

See [TODO.md](./TODO.md) for upcoming features:

- Auto-checkpoint (when session grows too large)
- Checkpoint diff comparison
- Checkpoint merge functionality
- Visual session timeline
- Cross-session search optimization

## License

Same as parent project.

## Contributing

See [design.md](./design.md) for contribution guidelines.

---

**Last Updated:** 2026-01-21
**Version:** 2.12
