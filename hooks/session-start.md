---
when: SessionStart
---

# SessionStart Hook - 任务管理系统初始化检测

## 功能描述

每次 Claude Code 启动时自动执行以下操作：
1. 检测项目是否已建立任务管理系统
2. 如果未建立，提示用户初始化
3. 如果 session.md 丢失，自动重建
4. 显示当前任务摘要

## 执行逻辑

### 步骤 0: 检测未处理的完成任务（新增）

**目的**: 检测已完成但未记录到文档的任务（例如通过 `mv` 命令移动的任务卡片）

**执行逻辑**:
```bash
# 1. 扫描 done/ 目录中的任务文件
find "$DOCS_ROOT/done" -name "task-*.md" -type f | while read task_file; do
  # 提取任务ID
  task_id=$(basename "$task_file" | sed 's/task-\([0-9]*\)-.*/task-\1/')

  # 检查是否已在 archive-index.md 中记录
  if ! grep -q "$task_id" "$DOCS_ROOT/done/archive-index.md"; then
    # 任务未记录，需要补充记录
    echo "UNRECORDED_TASK: $task_file"
  fi
done
```

**自动修复**:
```markdown
## 🔍 检测到未记录的完成任务

发现以下任务已完成但未更新到项目文档：

{{UNRECORDED_TASKS}}

正在自动更新文档...

✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 已更新 archive-index.md
```

**更新内容**:
1. **session.md**: 将任务添加到"上一个任务"表格
2. **current-sprint.md**: 更新任务状态为 ✅ 完成
3. **archive-index.md**: 添加任务归档记录

### 步骤 1: 检测项目状态

```bash
# 检查关键文件和目录
DOCS_ROOT="${docs_root:-docs}"  # 从配置读取，默认 "docs"

# 检查是否存在任务管理系统
if [ ! -d "$DOCS_ROOT/todo" ] || [ ! -d "$DOCS_ROOT/done" ]; then
  # 项目未初始化 - 提示用户
  return "INITIALIZE_NEEDED"
fi

if [ ! -f "$DOCS_ROOT/session.md" ]; then
  # session.md 丢失 - 尝试重建
  return "REBUILD_NEEDED"
fi

# 正常启动
return "SHOW_SUMMARY"
fi
```

### 步骤 2: 初始化新项目

**触发条件**: `$DOCS_ROOT/todo` 或 `$DOCS_ROOT/done` 不存在

**操作**:
```markdown
## 🚀 检测到新项目

此项目尚未建立任务管理系统。是否立即初始化？

**将会创建**:
- \`$DOCS_ROOT/session.md\` - 当前 Session 状态
- \`$DOCS_ROOT/todo/roadmap.md\` - 长期路线图
- \`$DOCS_ROOT/todo/current-sprint.md\` - 当前冲刺
- \`$DOCS_ROOT/done/archive-index.md\` - 归档索引
- \`$DOCS_ROOT/todo/backlog/\` - 任务卡片目录
- \`$DOCS_ROOT/done/YYYY-MM/\` - 归档目录（当前月份）

**初始化命令**: 使用 `Write` 工具创建以下文件：
1. 从模板生成 \`$DOCS_ROOT/session.md\`
2. 从模板生成 \`$DOCS_ROOT/todo/roadmap.md\`
3. 从模板生成 \`$DOCS_ROOT/todo/current-sprint.md\`
4. 从模板生成 \`$DOCS_ROOT/done/archive-index.md\`
5. 创建目录 \`$DOCS_ROOT/todo/backlog/\`
6. 创建目录 \`$DOCS_ROOT/done/$(date +%Y-%m)/\`
```

### 步骤 3: 自动重建 session.md

**触发条件**: `todo/` 和 `done/` 存在，但 `session.md` 不存在

**操作**:
```markdown
## ⚠️ session.md 丢失 - 自动重建中...

从以下来源重建：
1. \`$DOCS_ROOT/todo/current-sprint.md\` - 当前任务
2. \`$DOCS_ROOT/done/archive-index.md\` - 最近完成任务
3. Git 历史 - 最近活动（如果可用）

**重建命令**:
\`\`\`bash
# 提取当前任务
CURRENT_TASKS=$(grep -A 10 "本周任务" "$DOCS_ROOT/todo/current-sprint.md" | grep "🔄" | head -1)

# 提取最近完成任务
LAST_COMPLETED=$(tail -20 "$DOCS_ROOT/done/archive-index.md" | grep "✅" | head -1)

# 生成新 session.md
cat > "$DOCS_ROOT/session.md" << EOF
# 当前 Session 任务状态

**更新时间**: $(date +%Y-%m-%d %H:%M)
**Session ID**: session-$(date +%Y%m%d)-auto-rebuilt

## 🎯 当前任务

$CURRENT_TASKS

## ⏳ 上一个任务

$LAST_COMPLETED

---
EOF
\`\`\`

✅ session.md 已重建
```

### 步骤 4: 显示任务摘要

**触发条件**: 正常启动

**操作**:
```markdown
## 📋 当前 Session 状态

🎯 **当前任务**: 从 \`$DOCS_ROOT/session.md\` 提取
📅 **本周进度**: 从 \`$DOCS_ROOT/todo/current-sprint.md\` 计算

### 待办事项
- 从 session.md 提取"本 Session 任务队列"

### 最近完成
- ✅ 从 session.md 提取"上一个任务"

**详细信息**:
- 查看 \`$DOCS_ROOT/session.md\` - 完整 session 状态
- 查看 \`$DOCS_ROOT/todo/current-sprint.md\` - 本周任务
- 查看 \`$DOCS_ROOT/todo/roadmap.md\` - 长期路线图
```

**如果有未记录的任务**:
```markdown
## ⚠️ 发现未记录的完成任务

以下任务已完成但未更新到文档：

{{UNRECORDED_TASK_LIST}}

是否立即更新文档？建议运行以下命令：
1. 手动更新 session.md、current-sprint.md、archive-index.md
2. 或使用命令自动更新：claude-task-pilot --sync-completed-tasks
```

## 实现要点

### 1. 非侵入式检测
- 只读取文件，不修改任何内容（除非重建）
- 使用 `Read` 工具检查文件存在性

### 2. 错误处理（E1）
- session.md 缺失 → 自动重建
- 目录缺失 → 提示初始化
- 重建失败 → 显示警告，不影响正常使用

### 3. 性能优化
- 只读取必要的前 20 行
- 缓存文件内容，避免重复读取
- 总耗时 < 100ms

## 配置参数

从 `.claude/task-management.local.md` 读取：
- `docs_root`: 文档根目录（默认: "docs"）
- `session_max_lines`: session.md 最大行数（默认: 80）

## 测试场景

1. **新项目**: 完全空的 `docs/` 目录
2. **部分初始化**: 只有 `todo/` 或 `done/`
3. **文件丢失**: 有目录结构但缺 `session.md`
4. **正常启动**: 完整的文件结构
5. **损坏文件**: `session.md` 内容为空或格式错误
