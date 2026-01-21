# TODO: Claude Code Checkpoint Summary System

## Phase 1: Core Implementation (COMPLETED ✅)

- [x] Design Document v1.0
- [x] Design Document v2.0 (UUID format)
- [x] Design Document v2.1 (final UUID update)
- [x] Design Document v2.4 (removed /clear automation)
- [x] Design Document v2.5 (SKILL.md spec)
- [x] Design Document v2.6 (Key Design Decisions)
- [x] Design Document v2.7 (Fixed Session ID detection)
- [x] Design Document v2.9 (Lesson Learned: Keep It Simple)
- [x] Design Document v2.10 (Post-Clear Restoration)
- [x] Design Document v2.11 (MCP Unavailable Fallback)
- [x] Design Document v2.12 (Added reference.md Quick Reference)
- [x] SKILL.md v2.11 (Following sync-memora pattern - Step 0 MCP Readiness Check)
- [x] SKILL.md v2.12 (Simplified - Version History Navigator only, link to changelog.md)
- [x] reference.md v1.0 (Quick reference tables for MCP tools, error codes, troubleshooting)
- [x] Version History and Changelog structure
- [x] README.md documentation
- [x] TODO.md roadmap

---

## Phase 2: Implementation Tasks (IN PROGRESS 🚧)

### 2.1 Core Skills

#### /checkpoint Skill (SKILL.md - Claude Code CLI Spec)
- [x] Create `/checkpoint` SKILL.md file (v1.6.0)
  - [x] YAML frontmatter: name, description, allowed-tools, user-invocable
  - [x] Parse options: `--summary-only`, `--include-files`, `--tags`, `--title`
  - [x] Add list options: `--list`, `--parent`, `--all`
  - [x] Removed `--clear` option (infeasible - built-in commands not available via Skill tool)
  - [x] Fixed Session ID detection - from Claude Code CLI (<env> tags), NOT generated
- [ ] Implement `/checkpoint` skill functionality
  - [ ] Read current session metadata from environment/state
  - [ ] Generate checkpoint ID: `ccp-<uuid>-<seq>-<unix_ms>`
  - [ ] Analyze session content
    - [ ] Extract topics from conversation
    - [ ] Generate summary
    - [ ] List files referenced
    - [ ] Track tools used
  - [ ] Create observation in claude-mem
    - [ ] Master summary observation
    - [ ] Detailed observations (optional)
  - [ ] Update session state
  - [ ] Return checkpoint report to user
  - [ ] List checkpoints when --list option provided

### 2.2 Session State Management (MCP-Only)

- [ ] Session state storage via MCP (claude-mem observations)
  - [ ] Environment variables: `CLAUDE_SESSION_ID`, `CLAUDE_LAST_CHECKPOINT`, etc.
  - [ ] Observation-based storage (primary - all state in MCP)

- [ ] Session metadata tracking
  - [ ] session_id (UUID)
  - [ ] started_at (ISO 8601)
  - [ ] started_at_unix (milliseconds)
  - [ ] parent_session (UUID, optional)
  - [ ] checkpoint_count
  - [ ] last_checkpoint_id
  - [ ] working_directory
  - [ ] hostname
  - [ ] message_count
  - [ ] duration_seconds

### 2.3 Hook Policy Implementation (Optional)

#### guard_clear.py (Optional - MCP-based)
- [ ] Create guard_clear.py script
  - [ ] Location: `$CLAUDE_PROJECT_DIR/.claude/hooks/`
  - [ ] Use MCP search to verify checkpoint exists
  - [ ] Check for `/clear --force` flag FIRST
    - [ ] If `--force` detected: Allow with warning
    - [ ] Log force clear event to audit log
  - [ ] If prompt is `/clear` (no force) and no recent checkpoint:
    - [ ] Return: `{ "decision": "block", "reason": "Run /checkpoint before /clear. Use /clear --force to bypass." }`
  - [ ] Otherwise: Return `{ "decision": "allow" }`
  - [ ] Make executable: `chmod +x guard_clear.py`

#### settings.json Configuration
- [ ] Add UserPromptSubmit hook to settings.json
  ```json
  {
    "hooks": {
      "UserPromptSubmit": [
        {
          "hooks": [
            {
              "type": "command",
              "command": "python3 \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/guard_clear.py"
            }
          ]
        }
      ]
    }
  }
  ```

---

## Phase 3: Enhanced Features (FUTURE 🚀)

### 3.1 Auto-Checkpoint

- [ ] Detect session size threshold
  - [ ] Token count monitoring
  - [ ] Message count threshold
  - [ ] Duration-based triggers
- [ ] Auto-generate checkpoint when threshold reached
- [ ] Notify user about auto-checkpoint
- [ ] Configurable thresholds in settings

### 3.2 Checkpoint Diff

- [ ] Compare two checkpoints
  - [ ] Show differences in session state
  - [ ] Highlight new files/changes
  - [ ] Display progress between checkpoints
- [ ] Visual diff representation
- [ ] Merge functionality

### 3.3 Visual Session Timeline

- [ ] Generate ASCII/visual timeline of session
  - [ ] Show checkpoints as nodes
  - [ ] Show session transitions (/clear)
  - [ ] Display lineage tree
- [ ] Export timeline as image/diagram
- [ ] Timeline query by date range

### 3.4 Cross-Session Search

- [ ] Search across multiple sessions
- [ ] Filter by date range
- [ ] Filter by topics
- [ ] Unified results display
- [ ] Relevance ranking

### 3.5 Checkpoint Analytics

- [ ] Session statistics dashboard
  - [ ] Total checkpoints
  - [ ] Checkpoint frequency
  - [ ] Session duration analysis
  - [ ] Tools usage patterns
- [ ] Activity trends over time

---

## Phase 4: Integration & Testing (PENDING 📋)

### 4.1 Claude Code Integration

- [ ] Get Session ID from Claude Code
  - [ ] Environment variable: `$CLAUDE_SESSION_ID`
  - [ ] Hook context: `SessionStart.session_id`
  - [ ] Verify UUID v4 format

- [ ] SessionStart hook handling
  - [ ] Detect session start
  - [ ] Initialize session state
  - [ ] Handle `source: "startup"` vs `source: "clear"`

### 4.2 claude-mem Integration

- [ ] Observation creation
  - [ ] Use MCP tools directly
  - [ ] Follow template structure
  - [ ] Include all metadata fields
  - [ ] Handle errors gracefully

- [ ] Observation retrieval
  - [ ] Query patterns documented in design.md
  - [ ] Parse checkpoint observations
  - [ ] Display restored context

### 4.3 Testing

- [ ] Unit tests
  - [ ] Checkpoint ID generation
  - [ ] Session lineage tracking
  - [ ] UUID validation

- [ ] Integration tests
  - [ ] Full checkpoint creation flow
  - [ ] Session transition (/clear)
  - [ ] Context restoration
  - [ ] Hook policy enforcement

- [ ] Manual testing
  - [ ] Test with real Claude Code session
  - [ ] Verify checkpoint creation via MCP
  - [ ] Verify guard_clear.py blocking (if implemented)
  - [ ] Test restoration queries

---

## Phase 5: Documentation (ONGOING 📚)

- [x] README.md - Project overview
- [x] design.md - Technical specification v2.3
- [x] TODO.md - This file
- [ ] changelog.md - Version history
- [ ] API Documentation - For skill developers
- [ ] User Guide - Step-by-step usage
- [ ] Troubleshooting Guide - Common issues

---

## Phase 6: Maintenance & Optimization (FUTURE 🔧)

### 6.1 Performance

- [ ] Checkpoint creation optimization
- [ ] Query performance tuning
- [ ] Storage cleanup strategy
  - [ ] Old checkpoint archival
  - [ ] Observation cleanup
  - [ ] Session state cleanup

### 6.2 Error Handling

- [ ] Comprehensive error messages
- [ ] Retry logic for failures
- [ ] Fallback mechanisms
- [ ] User-friendly error recovery

### 6.3 Compatibility

- [ ] Backward compatibility check
- [ ] Migration from old format (if needed)
- [ ] Version field validation
- [ ] Format field validation

---

## Blocked Issues

Currently no blocked issues.

---

## Next Steps (Priority Order)

1. **IMMEDIATE** (This session):
   - ✅ Create SKILL.md following Claude Code CLI spec (v1.4.0)
   - ✅ Remove --clear option (infeasible)
   - ✅ Update design.md to v2.5
   - Test `/checkpoint` skill in Claude Code
   - Verify MCP tool calls work

2. **SHORT-TERM** (Next session):
   - Test with real Claude Code session
   - Verify all components work together
   - Document any issues found

3. **MEDIUM-TERM** (Future sessions):
   - Auto-checkpoint feature
   - Visual session timeline
   - Enhanced search capabilities

---

## Notes

- **Session ID Format**: Must use Claude Code native UUID (36 chars, UUID v4)
- **CRITICAL**: Session ID is PROVIDED by Claude Code CLI (<env> tags), NOT generated by skill
- **Checkpoint ID Format**: Must follow `ccp-<uuid>-<seq>-<unix_ms>` pattern
- **Timestamps**: Always use milliseconds precision for Unix timestamps
- **Version Field**: Always include `version: "2.3"` in metadata for compatibility
- **Design Change**: Removed `/session-info` skill - use `/checkpoint --list` instead
- **Design Change**: Removed checkpoint marker system - ALL state in MCP only
- **Design Change**: Using SKILL.md (uppercase) following Claude Code CLI spec with YAML frontmatter
- **Design Change**: Converted from .skill.md to SKILL.md (v1.4.0) to match official skill specification
- **Design Change (v2.7)**: Fixed Session ID detection - reads from Claude Code CLI environment, NO UUID generation
- **Design Change (v2.8)**: Simplified Session ID - AI knows its own Session ID, just read and use
- **Design Change (v2.9)**: Added Lesson Learned - Keep It Simple: answer may be right in front
- **Design Change (v2.10)**: Added Post-Clear Restoration design - /clear doesn't change Session ID
- **Design Change (v2.11)**: Added MCP Unavailable Fallback - graceful degradation when MCP tools unavailable
- **SKILL.md Change (v2.11)**: Following sync-memora pattern - added Step 0 MCP Readiness Check, Error Handling section
