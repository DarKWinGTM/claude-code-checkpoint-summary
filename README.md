# 🎯 Claude Code Checkpoint Summary System

<div align="center">

![Version: 2.12](https://img.shields.io/badge/version-2.12-blue)
![Status: Production Ready](https://img.shields.io/badge/status-production%20ready-brightgreen)
![License: MIT](https://img.shields.io/badge/license-MIT-yellow)

</div>

---

## 📖 Overview

ระบบ **checkpoint** สำหรับ Claude Code sessions ที่ช่วยให้:

| 🎯 Feature | 📝 Description |
|-----------|---------------|
| **Track History** | Session IDs แบบ UUID ที่ unique และ traceable |
| **Save Checkpoints** | Observations ใน claude-mem พร้อม metadata ครบถ้ม |
| **Restore Context** | กู้คืน session context จาก checkpoints ได้อย่างยิ่งขึ้น |
| **Trace Lineage** | ติดตาม session lineage ผ่าน `/clear` operations |

---

## 🚀 Quick Start

```bash
# Install: Add to ~/.claude/skills/checkpoint/SKILL.md

# Usage: Create checkpoint
/checkpoint --title "Feature Implementation Complete"

# List checkpoints
/checkpoint --list

# Restore latest checkpoint
/checkpoint --restore
```

---

## 🎨 Key Features

### 1. UUID-based Session Identification

```text
Session ID: 7670db3a-2057-406a-a109-afcedef1cb97
  ├─ Length: 36 characters (UUID v4)
  ├─ Entropy: 2^122 possibilities
  └─ Source: Claude Code native
```

### 2. Checkpoint Reference System

```text
Checkpoint ID: ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
  ├─ ccp: Checkpoint prefix
  ├─ session_uuid: Full UUID (36 chars)
  ├─ seq: Sequence (01, 02, 03...)
  └─ unix_ms: Timestamp (13 digits)
```

### 3. Session Lineage Tracking

```text
Session A (root)
├─ Checkpoint 1, 2
└─ /clear
   ↓
Session B (continuation)
├─ parent_session: Session A
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

---

## 📋 Version History

**Current Version:** v2.12 (2026-01-21)

| Version | Date | Key Changes |
|---------|------|-------------|
| 2.12 | 2026-01-21 | ✅ Added reference.md Quick Reference Documentation |
| 2.11 | 2026-01-21 | ✅ MCP Readiness Check & Fallback Mode |
| 2.10 | 2026-01-21 | ✅ Post-Clear Restoration design |
| 2.9 | 2026-01-21 | ✅ Lesson Learned - Keep It Simple |
| 2.8 | 2026-01-21 | ✅ Simplified Session ID (AI knows its own ID) |
| 2.1 | 2025-01-19 | ✅ UUID Format Update (Claude Code native) |

> 📜 **Full History:** [changelog.md](./changelog.md)

---

## 🏗️ Architecture

### Components

```
┌─────────────────────────────────────────────────┐
│  SKILL IMPLEMENTATION LAYER                    │
│  ├─ /checkpoint → SKILL.md instructions       │
│  ├─ Session ID from <env> (not generated)    │
│  ├─ Checkpoint count from MCP query            │
│  └─ AI-powered content analysis              │
├─────────────────────────────────────────────────┤
│  MCP TOOLS LAYER (Existing)                    │
│  ├─ mcp__plugin_claude-mem_memory_create     │
│  ├─ mcp__plugin_claude-mem_mem-search__search  │
│  └─ Other claude-mem MCP tools                 │
├─────────────────────────────────────────────────┤
│  STORAGE LAYER                                │
│  └─ claude-mem observations database           │
└─────────────────────────────────────────────────┘
```

### Data Flow

```
Session Start
  ↓
Claude Code generates UUID v4 (Session ID)
  ↓
User works...
  ↓
User runs: /checkpoint
  ↓
1. MCP Query: checkpoint count
2. Generate Checkpoint ID
3. AI analyzes session content
4. Create observation in claude-mem
  ↓
User runs: /clear
  ↓
New session continues (same UUID)
  ↓
Can restore: /checkpoint --restore
```

---

## 📚 Documentation

| File | Purpose |
|------|---------|
| **[design.md](./design.md)** | 📐 Complete technical specifications |
| **[changelog.md](./changelog.md)** | 📜 Version history and changes |
| **[skills/checkpoint/SKILL.md](./skills/checkpoint/SKILL.md)** | ⚙️ Checkpoint skill implementation |
| **[skills/checkpoint/reference.md](./skills/checkpoint/reference.md)** | 📚 Quick reference tables |
| **[TODO.md](./TODO.md)** | 📋 Development roadmap |

---

## 🎯 Problem Solved

### Issues Before

| ❌ Issue | 🔧 Solution |
|---------|-----------|
| Session ID ไม่ unique | UUID v4: 2^122 possibilities |
| ไม่มี timestamp ที่ชัดเจน | Unix milliseconds in ID |
| Session lineage ไม่ชัดเจน | parent_session metadata |
| Checkpoint ล่าสุดไม่ชัดเจน | MCP query with sorting |

### Solution Architecture

```
Session ID (UUID v4) → Checkpoint ID (ccp-<uuid>-<seq>-<unix_ms)
       ↓                         ↓
   Traceable                    Restorable
```

---

## 🛠️ Usage Examples

### Basic Checkpoint
```bash
/checkpoint
```

### Custom Title
```bash
/checkpoint --title "API Design Complete"
```

### Custom Tags
```bash
/checkpoint --tags "api,design,rest"
```

### List Checkpoints
```bash
/checkpoint --list
```

### List All (with parent session)
```bash
/checkpoint --list --all
```

### Restore Latest Checkpoint
```bash
/checkpoint --restore
```

---

## 📊 Project Status

**v2.12** - Production Ready ✅

| Feature | Status |
|---------|--------|
| UUID-based Session IDs | ✅ Implemented |
| Checkpoint Reference System | ✅ Implemented |
| Session Lineage Tracking | ✅ Implemented |
| Cross-Session Restoration | ✅ Implemented |
| MCP Readiness Check | ✅ Implemented (v2.11) |
| Fallback Mode | ✅ Implemented (v2.11) |
| Quick Reference Docs | ✅ Added (v2.12) |
| Rules-Compliant Structure | ✅ Verified (v2.12) |

---

## 🔗 Dependencies

| Dependency | Purpose |
|------------|---------|
| **Claude Code** | Native UUID generation |
| **claude-mem MCP** | Observation storage |
| **GitHub** | Repository hosting |

---

## 🚧 Future Enhancements

See [TODO.md](./TODO.md) for upcoming features:

- [ ] Auto-checkpoint (session size threshold)
- [ ] Checkpoint diff comparison
- [ ] Checkpoint merge functionality
- [ ] Visual session timeline
- [ ] Cross-session search optimization

---

## 📄 License

Same as parent project.

---

## 🤝 Contributing

See [design.md](./design.md) for contribution guidelines.

---

<div align="center">

**🌟 Star this repo on GitHub!**

[⭐ Star](https://github.com/DarKWinGTM/claude-code-checkpoint-summary/stargazers) |
[🐛 Report Issue](https://github.com/DarKWinGTM/claude-code-checkpoint-summary/issues) |
[📖 View Docs](https://github.com/DarKWinGTM/claude-code-checkpoint-summary/wiki)

</div>

---

<div align="center">

**📧 Contact:** [DarKWinGTM](https://github.com/DarKWinGTM)

---

**Last Updated:** 2026-01-21
**Version:** 2.12
**Repository:** https://github.com/DarKWinGTM/claude-code-checkpoint-summary

</div>
