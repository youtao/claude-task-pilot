---
when: PreToolWrite
ifTool: Write
---

# PreToolWrite Hook - 任务开始检测

## 功能描述

监控文件创建操作，当检测到新任务卡片创建时，自动更新相关文档：
- 更新 `session.md` "当前任务" 部分
- 更新 `current-sprint.md`，标记为 🔄 进行中

## 检测逻辑

### 任务卡片识别

**匹配模式**:
```
docs/todo/backlog/task-XXX-{description}.md
```

其中：
- `XXX` 是三位数字（如 002, 003）
- `{description}` 是简短的任务描述（kebab-case）

**正则表达式**:
```javascript
/^docs\/todo\/backlog\/task-\d{3}-[\w-]+\.md$/
```

### 触发条件

```javascript
// 在 PreToolUse hook 中
if (toolName === "Write" || toolName === "mcp__plugin_serena_serena__create_text_file") {
  const filePath = toolArgs.file_path || toolArgs.relative_path

  if (isTaskCard(filePath)) {
    const taskId = extractTaskId(filePath)  // 如 "task-002"
    const taskName = extractTaskName(filePath)  // 如 "tomato-data-collection"

    return {
      action: "TASK_STARTED",
      taskId,
      taskName,
      filePath
    }
  }
}
```

## 自动更新操作

### 步骤 1: 读取当前状态

```markdown
**操作**: 使用 `Read` 工具读取以下文件：
1. `$DOCS_ROOT/session.md` - 提取当前任务信息
2. `$DOCS_ROOT/todo/current-sprint.md` - 提取任务列表
```

### 步骤 2: 更新 session.md

**位置**: "## 🎯 当前任务" 表格

**更新内容**:
```markdown
| 任务ID | 描述 | 状态 | 负责模块 | 优先级 |
|--------|------|------|----------|--------|
| {{TASK_ID}} | {{TASK_NAME}} | 🔄 进行中 | {{MODULE}} | {{PRIORITY}} |
```

**提取信息**:
- 从新创建的任务卡片文件中提取：
  - `任务描述`: 文件名或内容中的 `##` 标题
  - `负责模块`: 从 `**模块**` 字段提取，默认 "未分类"
  - `优先级`: 从 `**优先级**` 字段提取，默认 "P1"

**实现方式**:
```javascript
// 使用 Edit 工具替换 "当前任务" 表格
Edit({
  file_path: "$DOCS_ROOT/session.md",
  old_string: currentTaskTable,
  new_string: newTaskTable
})
```

### 步骤 3: 更新 current-sprint.md

**位置**: "## 本周任务" 表格

**更新内容**:
```markdown
| task-002 | 收集番茄种植专业资料 | ⏳ 待开始 | 2026-01-31 | - |
```
↓
```markdown
| task-002 | 收集番茄种植专业资料 | 🔄 进行中 | 2026-01-31 | - |
```

**状态映射**:
- ⏳ 待开始 → 🔄 进行中
- 其他状态保持不变

### 步骤 4: 添加到 Session 任务队列

**位置**: `session.md` "## 📋 本 Session 任务队列"

**操作**:
```markdown
1. [ ] {{TASK_NAME}}
```

如果任务已存在，跳过此步骤。

## 错误处理

### E3. 目录不存在

**检测**: `backlog/` 目录不存在

**处理**:
```markdown
⚠️ 检测到任务卡片创建，但目录结构不完整

自动创建：
- \`$DOCS_ROOT/todo/backlog/\`

✅ 已修复，继续更新...
```

### E4. 任务卡片格式错误

**检测**: 文件名不符合 `task-XXX-*.md` 格式

**处理**:
```markdown
⚠️ 文件名格式不符合任务卡片规范

期望格式: \`task-XXX-description.md\` (如 task-002-tomato-data.md)
实际文件名: {{ACTUAL_FILENAME}}

建议：重命名文件以符合规范
```

## 配置参数

```yaml
docs_root: "docs"
session_max_lines: 80
task_pattern: "^task-\\d{3}-[\\w-]+\\.md$"
```

## 测试场景

1. **正常任务创建**: 创建 `task-003-tomato-variety.md`
2. **错误格式**: 创建 `tomato-task.md`（应警告）
3. **重复创建**: 已存在的任务ID（应提示）
4. **目录缺失**: `backlog/` 不存在（应自动创建）
5. **并发操作**: 同时创建多个任务（应按顺序处理）

## 性能考虑

- **只监控 `Write` 操作**: `Edit` 操作不触发
- **文件路径过滤**: 只处理 `backlog/` 目录
- **幂等性**: 重复执行不产生副作用
- **异步更新**: 不阻塞原文件创建操作

## 实现示例

```javascript
// 完整流程
async function onPreToolWrite(toolArgs) {
  const { file_path, content } = toolArgs

  // 1. 检查是否是任务卡片
  if (!isTaskCard(file_path)) {
    return null  // 不处理
  }

  // 2. 提取任务信息
  const taskId = extractTaskId(file_path)
  const taskInfo = parseTaskInfo(content)

  // 3. 更新 session.md
  await updateSessionMd(taskId, taskInfo)

  // 4. 更新 current-sprint.md
  await updateCurrentSprint(taskId)

  // 5. 记录日志
  log(`✅ 已标记任务 ${taskId} 为进行中`)

  return { updated: true }
}
```
