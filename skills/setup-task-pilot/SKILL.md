---
name: setup-task-pilot
description: This skill should be used when the user asks to "initialize task management", "setup project structure", "fix project structure", "repair project files", "create project documentation", or "prepare project for task tracking". Provides intelligent project initialization for Claude Task Pilot with smart file detection and interactive confirmation.
version: 1.3.0
---

# Claude Task Pilot Project Initialization Skill

## Overview

Initialize or repair project documentation structure for Claude Task Pilot. Handle mid-project plugin installation with smart file detection and interactive confirmation.

## Core Principles

1. **Hybrid Mode**: Skip existing files, create missing ones
2. **Smart Detection**: Auto-ask for small files (< 5 lines), skip rich content
3. **Interactive Confirmation**: Give users choice for critical files
4. **Safety First**: Never delete existing content, only create missing parts

## Standard Directory Structure

```
docs/
├── session.md              # 当前 Session 状态
├── todo/
│   ├── roadmap.md          # 长期路线图
│   ├── current-sprint.md   # 当前冲刺
│   └── backlog/            # 任务卡片目录
└── done/                   # 已完成任务归档
    ├── archive-index.md    # 归档索引
    └── YYYY-MM/            # 按月归档目录（当前月份）
```

## Initialization Workflow

### Step 1: Check Existing Structure

Use Bash tool to check:

```bash
# 检查目录
test -d docs && echo "EXISTS" || echo "MISSING"
test -d docs/todo && echo "EXISTS" || echo "MISSING"
test -d docs/todo/backlog && echo "EXISTS" || echo "MISSING"
test -d docs/done && echo "EXISTS" || echo "MISSING"

# 检查文件
test -f docs/session.md && echo "EXISTS" || echo "MISSING"
test -f docs/todo/roadmap.md && echo "EXISTS" || echo "MISSING"
test -f docs/todo/current-sprint.md && echo "EXISTS" || echo "MISSING"
test -f docs/done/archive-index.md && echo "EXISTS" || echo "MISSING"
```

### Step 2: Create Missing Directories

Create directories that don't exist:

```bash
mkdir -p docs/todo/backlog
mkdir -p docs/done/$(date +%Y-%m)
```

### Step 3: Smart File Processing

For each content file:

1. **File doesn't exist** → Create from template
2. **File exists** → Check line count:
   - < 5 lines → Use AskUserQuestion to ask whether to overwrite
   - ≥ 5 lines → Skip, preserve user content

**Check file line count**:

```bash
wc -l < docs/session.md 2>/dev/null || echo "0"
```

**AskUserQuestion example**:

```
Detected docs/session.md exists but has minimal content (3 lines).
This may indicate incomplete initialization.

Regenerate using standard template?
- Option A: Regenerate (overwrite existing content)
- Option B: Keep existing content
```

### Step 4: Generate Report

After initialization completes, display:

```markdown
## ✅ 初始化完成

**创建的目录**：
- docs/todo/
- docs/todo/backlog/
- docs/done/
- docs/done/2026-01/

**创建的文件**：
- docs/session.md
- docs/todo/roadmap.md
- docs/todo/current-sprint.md

**跳过的文件**（已存在且内容丰富）：
- docs/done/archive-index.md

**下一步**：
1. 在 docs/todo/backlog/ 创建你的第一个任务卡片
2. 插件会自动追踪任务状态变化
```

## Template File Locations

Use `${CLAUDE_PLUGIN_ROOT}` to reference templates:

- `${CLAUDE_PLUGIN_ROOT}/templates/session.md`
- `${CLAUDE_PLUGIN_ROOT}/templates/roadmap.md`
- `${CLAUDE_PLUGIN_ROOT}/templates/current-sprint.md`
- `${CLAUDE_PLUGIN_ROOT}/templates/archive-index.md`

## Special Scenarios

### Scenario 1: Fresh Project

No files or directories exist.
→ Silently create all content, no prompts needed.

### Scenario 2: Partial Initialization

Some files exist, some don't.
→ Create only missing parts, skip existing ones.

### Scenario 3: Failed Initialization

Files exist but have minimal content (< 5 lines).
→ Ask whether to regenerate.

### Scenario 4: Complete Project

All files exist with rich content.
→ Report "Project fully initialized", no action needed.

## Error Handling

### Permission Issues

If file creation fails, inform the user:

```markdown
❌ 创建文件失败：docs/session.md

可能原因：
- 目录权限不足
- 磁盘空间不足

建议：
1. 检查 docs/ 目录权限
2. 手动创建目录后再试
```

### Missing Template Files

If template files don't exist, use built-in default templates.

## Force Mode

When the user requests "force initialization" or explicitly asks to skip confirmations, bypass all AskUserQuestion prompts and create missing files directly.

## Best Practices

1. **Always show progress**: Tell the user what's happening
2. **Clear reporting**: Display what was created and what was skipped
3. **Protect user data**: Never delete or overwrite existing content without explicit user consent
4. **Provide next steps**: Guide users on how to start after initialization completes

## Example Dialogue

**User**: /setup-task-pilot

**Claude**:
```
🔍 Checking project structure...

✅ Created directory: docs/todo/backlog/
✅ Created directory: docs/done/2026-01/
✅ Created file: docs/session.md
✅ Created file: docs/todo/roadmap.md

⚠️ Detected docs/todo/current-sprint.md exists (12 lines)
   → Preserving existing content

🎉 Initialization complete!

Ready to create task cards:
1. Create task cards in docs/todo/backlog/
2. Plugin will automatically track task status
```

## Additional Resources

### Template Files
- **`${CLAUDE_PLUGIN_ROOT}/templates/`** - Plugin template directory
  - `session.md` - Session status template
  - `roadmap.md` - Roadmap template
  - `current-sprint.md` - Sprint template
  - `archive-index.md` - Archive index template
  - `task-complete.md` - Task completion report template

### Documentation
- **[README.md](../../../README.md)** - User documentation and quick start guide
- **[CLAUDE.md](../../../CLAUDE.md)** - Plugin architecture and design principles
