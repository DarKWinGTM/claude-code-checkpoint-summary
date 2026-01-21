# Checkpoint Reference Documentation

> **Full Design:** [../../design.md](../../design.md)
> **Main Skill:** [SKILL.md](SKILL.md)

---

## MCP Tool Quick Reference

### Claude-Mem MCP Tools Used by Checkpoint Skill

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `mcp__plugin_claude-mem_mem-search__search` | Query checkpoint count | `query`, `limit` |
| `mcp__plugin_claude-mem_mem-search__get_observations` | Fetch checkpoint details | `ids` (array) |
| `mcp__plugin_claude-mem_memory_create` | Create simple checkpoint | `content`, `metadata`, `tags` |
| `mcp__plugin_claude-mem_memory_create_rich` | Create checkpoint with files | `content`, `metadata`, `tags`, `attachments` |

> **See:** [design.md - MCP Tools Layer](../../design.md#mcp-tools-layer-existing---claude-mem)

---

## Checkpoint ID Format Quick Reference

### Format Pattern
```
ccp-<session_uuid>-<seq>-<unix_ms>
```

### Components
| Component | Format | Example |
|-----------|--------|---------|
| Prefix | `ccp` | `ccp` |
| Session ID | UUID v4 (36 chars) | `7670db3a-2057-406a-a109-afcedef1cb97` |
| Sequence | Zero-padded (01-99) | `01`, `02`, `03` |
| Timestamp | Unix milliseconds (13 digits) | `1737196625000` |

### Example
```
ccp-7670db3a-2057-406a-a109-afcedef1cb97-01-1737196625000
```

> **See:** [design.md - Appendix A: ID Formats Reference](../../design.md#appendix-a-id-formats-reference)

---

## Command Options Quick Reference

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--title <title>` | string | Auto-generated | Custom checkpoint title |
| `--tags <tags>` | string | Auto-extracted | Custom comma-separated tags |
| `--summary-only` | flag | false | Only create master summary |
| `--include-files` | flag | false | Include file change details |
| `--list` | flag | false | List checkpoints (no creation) |
| `--parent` | flag | false | Include parent session in list |
| `--all` | flag | false | List entire session lineage |
| `--restore` | flag | false | Restore latest checkpoint |

> **See:** [design.md - API Specification](../../design.md#api-specification)

---

## Error Codes Quick Reference

| Code | Description | Quick Fix |
|------|-------------|-----------|
| `SESS_001` | Cannot generate session ID | Session ID from `<env>` - check Claude Code CLI |
| `SESS_002` | Session metadata corrupted | Reinitialize session |
| `CKPT_001` | Checkpoint creation failed | Check claude-mem MCP server status |
| `CKPT_002` | Checkpoint ID collision | Regenerate with new timestamp |
| `META_001` | Cannot save session metadata | Check disk permissions |

> **See:** [design.md - Appendix B: Error Codes](../../design.md#appendix-b-error-codes)

---

## MCP Configuration Quick Reference

### Required MCP Server

**Claude-Mem MCP** (Required)
```json
{
  "mcpServers": {
    "claude-mem": {
      "command": "npx",
      "args": ["-y", "@darkwingtm/claude-mem"]
    }
  }
}
```

### Database Location
- **Path:** `~/.claude-mem/` (default)
- **Format:** SQLite database
- **Verification:** MCP tool returns success if accessible

> **See:** [design.md - Environment Variables](../../design.md#environment-variables)

---

## Troubleshooting Quick Fixes

### MCP Connection Failed
```
❌ Error: Claude-Mem MCP not available
```

**Quick Steps:**
1. Check if claude-mem MCP server is running
2. Verify configuration in `~/.claude/settings.json`
3. Ensure database exists at `~/.claude-mem/`
4. Test with: `/checkpoint` (should show MCP Status Check)

---

### No Observations Found
```
ℹ️ No checkpoints found for this session.
```

**Quick Steps:**
1. This is normal for first checkpoint
2. Proceeding with sequence 1
3. First checkpoint will be created

---

### Checkpoint Creation Failed
```
❌ Error: Checkpoint creation failed
```

**Quick Steps:**
1. Check claude-mem MCP server status
2. Verify MCP configuration in `~/.claude/settings.json`
3. Ensure database exists and is writable
4. Check available disk space

---

### Session ID Not Found
```
❌ Error: Cannot determine session ID
```

**Quick Steps:**
1. Session ID should be in `<env>` tags
2. Check Claude Code CLI is running properly
3. Verify session started correctly

> **See:** [design.md - Error Handling](../../design.md#error-handling)

---

## Search Query Patterns

| Purpose | Query Pattern | Example |
|---------|---------------|---------|
| Latest checkpoint | `checkpoint session:<uuid> latest` | `checkpoint session:7670db3a... latest` |
| All session checkpoints | `checkpoint session:<uuid>` | `checkpoint session:7670db3a...` |
| Specific checkpoint | `checkpoint:<checkpoint_id>` | `checkpoint:ccp-7670db3a...-01-...` |
| Session lineage | `session lineage:<root_uuid>` | `session lineage:7670db3a...` |

> **See:** [design.md - Retrieval Mechanism](../../design.md#retrieval-mechanism)

---

## Metadata Schema Quick Reference

### Required Fields (v2.3+)
```json
{
  "version": "2.3",
  "format": "claude-code-checkpoint",
  "checkpoint_id": "ccp-<uuid>-<seq>-<unix_ms>",
  "session_id": "<uuid>",
  "sequence": 1,
  "type": "checkpoint",
  "created_at": "ISO_8601_TIMESTAMP",
  "created_at_unix": 1234567890000
}
```

### Optional Fields
```json
{
  "parent_session": null,
  "topics": ["topic1", "topic2"],
  "files": ["file1", "file2"],
  "tools_used": ["Read", "Write"],
  "working_directory": "/path/to/project"
}
```

> **See:** [design.md - Metadata Schema](../../design.md#metadata-schema)

---

## Version History

| Version | Date | Changes | Session ID |
|---------|------|---------|------------|
| 1.0 | 2026-01-21 | Initial reference documentation | a77b77ae-ef2a-49f6-93d9-f78c8ac2d2f7 |

---

> **For complete design documentation:** [../../design.md](../../design.md)
> **For skill implementation:** [SKILL.md](SKILL.md)
