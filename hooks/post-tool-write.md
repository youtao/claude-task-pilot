---
when: PostToolWrite
ifTool: Write|Edit|mcp__plugin_serena_serena__replace_content|Bash
---

# PostToolWrite Hook - 任务完成与规范守护

## 功能描述

监控文件操作，实现三大功能：
1. **任务完成检测**: 检测任务归档操作，自动更新文档
2. **文档规范守护**: 确保 session.md 不超过行数限制
3. **设计文档自动转换**: 检测设计文档创建，自动生成任务卡片

## 监听工具说明

- **Write/Edit/replace_content**: 文件写入操作
- **Bash**: Shell 命令执行（用于检测 `mv` 等文件移动命令）

---

## 功能 1: 任务完成检测

### 检测方式

#### 方式 A: 文件写入操作

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

#### 方式 B: Bash 命令移动（新增）

**检测逻辑**:
```javascript
// 检测 Bash 命令中的文件移动操作
if (toolName === "Bash") {
  const command = toolArgs.command

  // 匹配 mv 命令移动任务卡片
  const mvPattern = /mv\s+['"]?([^'"]+task-\d{3}-[\w-]+\.md)['"]?\s+['"]?([^'"]+done\/\d{4}-\d{2}\/)/
  const match = command.match(mvPattern)

  if (match) {
    const sourcePath = match[1].replace(/^['"]|['"]$/g, '')
    const targetPath = match[2].replace(/^['"]|['"]$/g, '')

    // 标准化路径
    if (!sourcePath.startsWith('docs/')) {
      sourcePath = `docs/${sourcePath}`
    }
    if (!targetPath.startsWith('docs/')) {
      targetPath = `docs/${targetPath}`
    }

    return { action: "TASK_COMPLETED", sourcePath, targetPath, tool: "Bash" }
  }
}
```

**支持的命令格式**:
```bash
# 格式 1: 直接路径
mv docs/todo/backlog/task-001-feature.md docs/done/2026-02/

# 格式 2: 引号包裹
mv "docs/todo/backlog/task-001-feature.md" "docs/done/2026-02/"

# 格式 3: 相对路径
mv docs/todo/backlog/task-001-feature.md docs/done/2026-02/

# 格式 4: Git mv
git mv docs/todo/backlog/task-001-feature.md docs/done/2026-02/
```

#### 方式 C: 状态标记（辅助方式）

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

## 功能 3: 设计文档自动转换

### 检测方式

**检测逻辑**:
```javascript
// 检测设计文档创建
if (filePath.match(/^docs\/plans\/\d{4}-\d{2}-\d{2}-.+-design\.md$/)) {
  return { action: "DESIGN_DOC_CREATED", filePath }
}
```

**触发条件**:
- 文件路径: `docs/plans/YYYY-MM-DD-<topic>-design.md`
- 操作: `Write` 或 `Edit` 工具创建/修改文件

### 自动转换流程

#### 步骤 1: 读取配置

```javascript
// 读取项目配置
const config = readConfig('.claude/claude-task-pilot.local.md')

// 检查是否启用自动转换
if (!config.auto_convert_designs) {
  log('ℹ️ 设计文档自动转换已禁用')
  log('提示: 使用 /convert-design-to-tasks 手动转换')
  return null
}
```

#### 步骤 2: 验证设计文档

```javascript
// 检查文件是否存在
if (!fileExists(filePath)) {
  log('⚠️ 设计文档不存在')
  return null
}

// 读取文件内容
const content = readFile(filePath)

// 检查内容是否有效
if (!content || content.length < 100) {
  log('⚠️ 设计文档内容过少，跳过自动转换')
  return null
}
```

#### 步骤 3: 询问用户（可选）

```javascript
// 根据配置决定是否询问
if (config.prompt_before_convert) {
  const basename = path.basename(filePath)
  const shouldConvert = await askUser(
    `检测到设计文档: ${basename}\n` +
    `是否自动生成任务卡片？ (Y/n)`
  )

  if (!shouldConvert) {
    log('ℹ️ 用户取消自动转换')
    log('提示: 稍后可使用 /convert-design-to-tasks 手动转换')
    return null
  }
}
```

#### 步骤 4: 调用转换 Agent

```javascript
// 调用 design-to-tasks agent
log('📋 正在分析设计文档...')

const result = await runAgent('design-to-tasks', {
  designDoc: filePath,
  mode: 'auto',  // 自动模式
  verbose: false
})

if (result.success) {
  log(`✅ 已创建 ${result.taskCount} 个任务卡片`)
  log(`任务ID: ${result.taskIds.join(', ')}`)

  // 显示简短摘要
  if (result.summary) {
    log('\n' + result.summary)
  }
} else {
  log(`❌ 转换失败: ${result.error}`)
  if (result.suggestion) {
    log(`建议: ${result.suggestion}`)
  }
}
```

### 配置参数

**`.claude/claude-task-pilot.local.md`**:
```yaml
---
# 设计文档自动转换配置
auto_convert_designs: true        # 启用自动转换（默认）
prompt_before_convert: false      # 不询问，直接生成（默认）
design_docs_path: "docs/plans"    # 设计文档路径
design_docs_pattern: "^docs/plans/\\d{4}-\\d{2}-\\d{2}-.+-design\\.md$"

# 任务生成配置
default_priority: "P1"            # 默认优先级
default_milestone: "Current Sprint"
task_complexity_threshold: 4      # 小时数，超过则拆分任务
---
```

### 错误处理

#### E1. backlog 目录不存在

```javascript
if (!dirExists(`${DOCS_ROOT}/todo/backlog`)) {
  log('⚠️ 任务目录未初始化')

  // 自动创建
  createDir(`${DOCS_ROOT}/todo/backlog`)
  createDir(`${DOCS_ROOT}/todo/current-sprint.md`)
  log('✅ 已自动创建任务目录')
}
```

#### E2. Agent 调用失败

```javascript
if (!result.success) {
  log(`❌ 设计文档转换失败`)
  log(`错误: ${result.error}`)

  // 提供恢复建议
  log('\n恢复选项:')
  log('1. 手动运行: /convert-design-to-tasks')
  log('2. 检查设计文档格式')
  log('3. 查看错误日志')

  return null
}
```

#### E3. 设计文档已处理

```javascript
// 检查设计文档中是否已有处理标记
if (content.includes('<!-- TASKS_GENERATED:')) {
  log('⚠️ 此设计文档已生成过任务卡片')

  if (config.prompt_before_convert) {
    const shouldRegenerate = await askUser('是否重新生成？ (Y/n)')
    if (!shouldRegenerate) {
      return null
    }
  }
}
```

### 测试场景

1. **自动触发**: 创建设计文档 → Hook 检测 → 询问确认 → 自动转换
2. **配置禁用**: auto_convert_designs=false → Hook 不触发
3. **跳过确认**: prompt_before_convert=false → 直接转换
4. **用户取消**: 用户选择"否" → 不转换，提示手动命令
5. **转换失败**: Agent 返回错误 → 显示错误信息和恢复建议

---

## 配置参数

```yaml
docs_root: "docs"
session_max_lines: 80  # session.md 最大行数
archive_threshold: 10  # 超过限制后多归档的行数

# 设计文档自动转换
auto_convert_designs: true        # 启用自动转换（默认）
prompt_before_convert: false      # 不询问，直接生成（默认）
design_docs_path: "docs/plans"
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

### 设计文档自动转换
1. **自动触发**: 创建设计文档 → Hook 检测 → 自动转换
2. **配置禁用**: auto_convert_designs=false → 不触发
3. **用户取消**: 询问时选择"否" → 不转换
4. **转换失败**: 显示错误和建议

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
  const { file_path, result, tool } = toolArgs

  // 功能 1: 任务完成检测
  const completionInfo = detectTaskCompletion(toolArgs)
  if (completionInfo) {
    await handleTaskCompletion(completionInfo)
  }

  // 功能 2: 文档规范守护
  if (file_path === `${DOCS_ROOT}/session.md`) {
    await checkSessionMdLimit()
  }

  // 功能 3: 设计文档自动转换
  if (isDesignDocCreation(file_path)) {
    await handleDesignDocCreation(file_path)
  }

  return null
}

function detectTaskCompletion(toolArgs) {
  const { tool, file_path, command } = toolArgs

  // 检测方式 1: 文件写入操作
  if ((tool === "Write" || tool === "Edit" || tool === "mcp__plugin_serena_serena__replace_content") &&
      file_path) {
    if (isTaskCompletionByFileWrite(file_path)) {
      return extractTaskInfoFromFile(file_path)
    }
  }

  // 检测方式 2: Bash 命令移动
  if (tool === "Bash" && command) {
    const mvInfo = parseMvCommand(command)
    if (mvInfo) {
      return mvInfo
    }
  }

  return null
}

function parseMvCommand(command) {
  // 匹配 mv/git mv 命令
  const patterns = [
    // 标准格式: mv source target
    /(?:git\s+mv|mv)\s+(['"]?)([^'"]+task-\d{3}-[\w-]+\.md)\1\s+(['"]?)([^'"]+done\/\d{4}-\d{2}\/(?:[^'"]+\/?))?\3/,
    // 简化格式: mv task-XXX.md done/
    /(?:git\s+mv|mv)\s+(task-\d{3}-[\w-]+\.md)\s+(done\/\d{4}-\d{2}\/?)/
  ]

  for (const pattern of patterns) {
    const match = command.match(pattern)
    if (match) {
      let sourcePath = match[2]
      let targetDir = match[4]

      // 标准化路径
      if (!sourcePath.startsWith('docs/')) {
        sourcePath = `docs/todo/backlog/${sourcePath}`
      }
      if (!targetDir.startsWith('docs/')) {
        targetDir = `docs/${targetDir}`
      }

      // 提取文件名
      const fileName = sourcePath.split('/').pop()
      const targetPath = targetDir.endsWith('/')
        ? `${targetDir}${fileName}`
        : `${targetDir}/${fileName}`

      return {
        action: "TASK_COMPLETED",
        sourcePath,
        targetPath,
        tool: "Bash",
        taskId: extractTaskIdFromPath(sourcePath)
      }
    }
  }

  return null
}

function extractTaskIdFromPath(path) {
  const match = path.match(/task-(\d{3})-[\w-]+\.md/)
  return match ? `task-${match[1]}` : null
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

async function handleDesignDocCreation(filePath) {
  // 1. 读取配置
  const config = readConfig()

  if (!config.auto_convert_designs) {
    return null  // 配置禁用，不触发
  }

  // 2. 验证文件
  if (!fileExists(filePath)) {
    log('⚠️ 设计文档不存在')
    return null
  }

  // 3. 询问用户（可选）
  if (config.prompt_before_convert) {
    const basename = path.basename(filePath)
    const shouldConvert = await askUser(
      `检测到设计文档: ${basename}\n是否自动生成任务卡片？ (Y/n)`
    )

    if (!shouldConvert) {
      log('ℹ️ 用户取消自动转换')
      log('提示: 使用 /convert-design-to-tasks 手动转换')
      return null
    }
  }

  // 4. 调用转换 Agent
  try {
    log('📋 正在分析设计文档...')

    const result = await runAgent('design-to-tasks', {
      designDoc: filePath,
      mode: 'auto'
    })

    if (result.success) {
      log(`✅ 已创建 ${result.taskCount} 个任务卡片`)
      log(`任务ID: ${result.taskIds.join(', ')}`)
    } else {
      log(`❌ 转换失败: ${result.error}`)
      if (result.suggestion) {
        log(`建议: ${result.suggestion}`)
      }
    }
  } catch (error) {
    log(`❌ Agent 调用失败: ${error.message}`)
  }
}

function isDesignDocCreation(filePath) {
  return filePath.match(/^docs\/plans\/\d{4}-\d{2}-\d{2}-.+-design\.md$/)
}
```
