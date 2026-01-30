---
when: PostToolWrite
ifTool: Write|Edit|mcp__plugin_serena_serena__replace_content
---

# PostToolWrite Hook - 任务完成与规范守护

## 功能描述

监控文件操作，实现两大功能：
1. **任务完成检测**: 检测任务归档操作，自动更新文档
2. **文档规范守护**: 确保 session.md 不超过行数限制

---

## 功能 1: 任务完成检测

### 检测方式

#### 方式 A: 文件移动（主要方式）

**检测逻辑**:
```javascript
// 检测到从 backlog/ 移动到 done/YYYY-MM/
if (sourcePath.match(/todo\/backlog\/task-\d{3}-[\w-]+\.md$/) &&
    targetPath.match(/done\/\d{4}-\d{2}\/.+/)) {
  return { action: "TASK_COMPLETED", taskId, sourcePath, targetPath }
}
```

**触发条件**:
- 源文件: `docs/todo/backlog/task-XXX-*.md`
- 目标文件: `docs/done/YYYY-MM/*.md`

#### 方式 B: 状态标记（辅助方式）

**检测逻辑**:
```javascript
// 检测到 backlog/ 文件中添加完成标记
const content = readFile(filePath)
if (content.includes("完成时间:") || content.includes("状态: ✅")) {
  return { action: "TASK_MARKED_COMPLETE", taskId, filePath }
}
```

### 自动归档流程

#### 步骤 1: 提取完成信息

```markdown
**从任务卡片文件中提取**:
1. **任务ID**: 如 "task-002"
2. **任务名称**: 文件名或标题
3. **完成时间**: 文件中 "完成时间" 字段，或当前时间
4. **关键成果**: 文件中 "## ✅ 完成内容" 部分
5. **技术决策**: 文件中 "## 💡 技术决策" 部分
```

#### 步骤 2: 创建完成报告

**文件路径**: `$DOCS_ROOT/done/YYYY-MM/task-XXX-{description}.md`

**如果文件已存在（E4）**:
```javascript
// 添加时间戳后缀
const timestamp = new Date().toISOString().replace(/[:.]/g, "-").slice(0, 19)
targetPath = `$DOCS_ROOT/done/YYYY-MM/task-XXX-${timestamp}.md`
```

**模板**: 使用 `task-complete.template`，填充提取的信息

#### 步骤 3: 更新 session.md

**位置 1**: "## 🎯 当前任务" → 清空或移到"上一个任务"

**操作**:
```markdown
## 🎯 当前任务

（清空或显示"暂无"）
```

**位置 2**: "## ⏳ 上一个任务" → 添加刚完成的任务

```markdown
| 任务ID | 描述 | 完成时间 | 归档位置 |
|--------|------|----------|----------|
| task-002 | 收集番茄种植专业资料 | 2026-01-30 | done/2026-01/task-002-tomato-data.md |
```

#### 步骤 4: 更新 current-sprint.md

**操作**:
```markdown
将任务状态从 🔄 进行中 改为 ✅ 完成

| task-002 | 收集番茄种植专业资料 | ✅ 完成 | 2026-01-31 | 2026-01-30 |
```

#### 步骤 5: 更新 archive-index.md

**位置**: `$DOCS_ROOT/done/archive-index.md`

**操作**:
```markdown
在对应月份的表格中添加一行：

| 2026-01-30 | task-002: 收集番茄种植专业资料 | 建立番茄种植知识库基础 | task-002-tomato-data.md |
```

#### 步骤 6: 智能推荐（F6）

**触发**: 任务完成后自动调用 `task-suggester` agent

**调用方式**:
```javascript
// 在 Hook 中调用 Agent
const result = await runAgent("task-suggester", {
  context: {
    currentProject: {
      name: getProjectName(),
      docsRoot: DOCS_ROOT
    },
    completedTasks: getCompletedTasks(),
    backlogTasks: getBacklogTasks(),
    currentMilestone: getCurrentMilestone()
  }
})

// Agent 返回推荐结果
if (result.success) {
  displayRecommendations(result.recommendations)
}
```

**输出**:
```markdown
## 🎯 建议的下一个任务

[由 task-suggester agent 生成]
```

---

## 功能 2: 文档规范守护

### 检测对象

**目标文件**: `$DOCS_ROOT/session.md`

**触发时机**: 任何修改 `session.md` 的 `Write` 或 `Edit` 操作

### 检测逻辑

```javascript
// 在 PostToolUse 中
if (filePath === `${DOCS_ROOT}/session.md`) {
  const lines = countLines(filePath)

  if (lines > SESSION_MAX_LINES) {  // 默认 80
    return { action: "ARCHIVE_NEEDED", lineCount: lines }
  }
}
```

### 自动归档流程

#### 步骤 1: 计算需要归档的内容

```javascript
const overflowLines = lines - SESSION_MAX_LINES
const archiveCount = overflowLines + 10  // 多归档 10 行，避免频繁触发

// 从文件顶部提取需要归档的行
const contentToArchive = readFirstLines(filePath, archiveCount)
const remainingContent = readFromLine(filePath, archiveCount)
```

#### 步骤 2: 创建归档文件

**文件路径**:
```
$DOCS_ROOT/done/YYYY-MM/session-archive-YYYYMMDD-HHMMSS.md
```

**内容结构**:
```markdown
# Session Archive - {{DATE}}

此文件从 session.md 自动归档，因超过行数限制。

---

## 归档内容

{{ARCHIVED_CONTENT}}

---

**归档时间**: {{TIMESTAMP}}
**归档原因**: 超过 {{SESSION_MAX_LINES}} 行限制
```

#### 步骤 3: 更新 session.md

**操作**: 用 `Write` 工具覆盖文件

```markdown
# 当前 Session 任务状态

**更新时间**: {{CURRENT_DATE}}
**说明**: 历史内容已归档到 done/YYYY-MM/session-archive-*.md

## 🎯 当前任务

{{REMAINING_CONTENT}}
```

#### 步骤 4: 更新 archive-index.md

**添加记录**:
```markdown
| 2026-01-30 | Session 归档 | session.md 超过 80 行限制 | session-archive-20260130.md |
```

---

## 错误处理

### E1. done/YYYY-MM/ 目录不存在

**处理**:
```javascript
const currentMonth = new Date().toISOString().slice(0, 7)  // "2026-01"
const dirPath = `${DOCS_ROOT}/done/${currentMonth}`

if (!dirExists(dirPath)) {
  createDir(dirPath)
  log(`🔧 已自动创建目录: ${dirPath}`)
}
```

### E2. 归档文件冲突

**场景**: `done/2026-01/task-002.md` 已存在

**处理**:
```javascript
// 方案 1: 添加时间戳
const timestamp = Date.now().toString()
targetPath = `${dirPath}/task-002-${timestamp}.md`

// 方案 2: 添加序号
let counter = 1
while (fileExists(targetPath)) {
  targetPath = `${dirPath}/task-002-v${counter}.md`
  counter++
}
```

**推荐**: 使用时间戳，更明确

### E3. archive-index.md 损坏

**检测**: 文件不存在或格式错误

**处理**:
```javascript
if (!fileExists(`${DOCS_ROOT}/done/archive-index.md`)) {
  // 从模板重新创建
  createFromTemplate("archive-index.template")
  log("⚠️ archive-index.md 已从模板重建")
}
```

---

## 配置参数

```yaml
docs_root: "docs"
session_max_lines: 80  # session.md 最大行数
archive_threshold: 10  # 超过限制后多归档的行数
```

---

## 测试场景

### 任务完成检测
1. **正常归档**: 移动 `task-002.md` 到 `done/2026-01/`
2. **文件冲突**: 目标文件已存在（应添加时间戳）
3. **目录缺失**: `done/2026-01/` 不存在（应自动创建）
4. **状态标记**: backlog/ 文件中添加"完成时间"字段

### 文档规范守护
1. **超过行数**: session.md 达到 90 行（应归档前 20 行）
2. **刚好达标**: session.md 正好 80 行（不触发归档）
3. **频繁超限**: 连续多次超限（应递增归档）
4. **空文件**: session.md 内容为空（应从模板重建）

---

## 性能优化

- **只检查 session.md**: 其他文件不检查行数
- **异步归档**: 不阻塞原操作
- **批量更新**: 多个任务完成时，批量更新索引
- **缓存文件内容**: 避免重复读取

---

## 实现示例

```javascript
async function onPostToolWrite(toolArgs) {
  const { file_path, result } = toolArgs

  // 功能 1: 任务完成检测
  if (isTaskCompletion(file_path)) {
    await handleTaskCompletion(file_path)
  }

  // 功能 2: 文档规范守护
  if (file_path === `${DOCS_ROOT}/session.md`) {
    await checkSessionMdLimit()
  }

  return null
}

async function handleTaskCompletion(filePath) {
  // 1. 提取任务信息
  const taskInfo = extractTaskInfo(filePath)

  // 2. 创建完成报告
  await createCompletionReport(taskInfo)

  // 3. 更新 session.md
  await updateSessionMd("TASK_COMPLETED", taskInfo)

  // 4. 更新 current-sprint.md
  await updateCurrentSprint("TASK_COMPLETED", taskInfo.taskId)

  // 5. 更新 archive-index.md
  await updateArchiveIndex(taskInfo)

  // 6. 调用智能推荐
  await suggestNextTask()

  log(`✅ 任务 ${taskInfo.taskId} 已归档`)
}

async function checkSessionMdLimit() {
  const lines = countLines(`${DOCS_ROOT}/session.md`)

  if (lines > SESSION_MAX_LINES) {
    await archiveSessionMd(lines)
    log(`📦 session.md 已归档（${lines} 行 > ${SESSION_MAX_LINES} 行）`)
  }
}
```
