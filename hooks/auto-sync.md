---
when: PostToolWrite
ifTool: Write|Edit|Bash
---

# Auto-Sync Hook - 自动文档同步

## 功能描述

在用户进行文件操作后，自动检测并同步文档，确保所有项目文档保持一致，无需手动运行 `/sync-progress` 命令。

---

## 触发时机

以下情况会触发自动同步检测：

1. **编辑任务卡片** (Edit/Write)
   - 编辑 `docs/todo/backlog/task-*.md`
   - 编辑 `docs/done/**/task-*.md`

2. **编辑关键文档** (Edit/Write)
   - 编辑 `docs/session.md`
   - 编辑 `docs/todo/current-sprint.md`
   - 编辑 `docs/todo/roadmap.md`

3. **执行关键命令** (Bash)
   - 移动/删除任务相关文件
   - 修改任务相关内容

---

## 自动同步逻辑

### 步骤 1: 判断是否需要同步

```javascript
// 检查配置
const config = readConfig('.claude/claude-task-pilot.local.md')

// 如果禁用自动同步，跳过
if (config.auto_sync === false) {
  return null
}

// 检查是否是任务相关的文件操作
const isTaskRelated = isTaskFileOperation(toolArgs)

if (!isTaskRelated) {
  return null  // 不是任务相关操作，跳过
}
```

### 步骤 2: 检测不一致

```javascript
// 快速检测（不扫描所有文件，只检查关键指标）
const issues = []

// 检查 1: 编辑了 backlog 中的任务卡片
if (filePath.match(/todo\/backlog\/task-\d{3}-[\w-]+\.md$/)) {
  const content = await readFile(filePath, 'utf-8')

  // 检查是否添加了完成标记
  if (content.includes('**完成时间**') || content.includes('状态: ✅')) {
    issues.push({
      type: 'TASK_COMPLETED_NOT_ARCHIVED',
      file: filePath,
      message: '任务已完成但未归档',
      autoFix: true
    })
  }
}

// 检查 2: 编辑了关键文档
if (['session.md', 'current-sprint.md', 'roadmap.md'].includes(path.basename(filePath))) {
  issues.push({
    type: 'KEY_DOC_UPDATED',
    file: filePath,
    message: '关键文档已更新，建议同步其他文档',
    autoFix: true
  })
}

// 如果没有发现问题，不需要同步
if (issues.length === 0) {
  return null
}
```

### 步骤 3: 决定是否自动修复

```javascript
// 根据配置决定是否自动修复
let shouldAutoFix = false

if (config.auto_sync_mode === 'auto') {
  // 完全自动模式，不询问
  shouldAutoFix = true
} else if (config.auto_sync_mode === 'prompt') {
  // 提示模式，询问用户
  const message = generatePromptMessage(issues)
  const userChoice = await askUser(message, { default: true })
  shouldAutoFix = userChoice
} else if (config.auto_sync_mode === 'silent') {
  // 静默模式，只修复明显的问题
  shouldAutoFix = issues.every(i => i.autoFix === true)
}

if (!shouldAutoFix) {
  return null
}
```

### 步骤 4: 执行快速同步

```javascript
// 执行轻量级同步（不扫描所有文件）
console.log('🔄 检测到文档不一致，自动同步中...\n')

// 1. 修复已完成的任务
for (const issue of issues) {
  if (issue.type === 'TASK_COMPLETED_NOT_ARCHIVED') {
    const sourcePath = issue.file
    const currentMonth = new Date().toISOString().slice(0, 7)
    const archiveDir = `docs/done/${currentMonth}`
    const fileName = path.basename(sourcePath)
    const targetPath = `${archiveDir}/${fileName}`

    // 创建归档目录
    await fs.mkdir(archiveDir, { recursive: true })

    // 移动文件
    const content = await readFile(sourcePath, 'utf-8')
    await writeFile(targetPath, content, 'utf-8')
    await fs.unlink(sourcePath)

    console.log(`✅ 已归档: ${fileName}`)
  }
}

// 2. 更新 session.md（如果相关）
if (issues.some(i => i.type === 'TASK_COMPLETED_NOT_ARCHIVED')) {
  await updateSessionMd()
  console.log('✅ 已更新 session.md')
}

// 3. 更新 current-sprint.md（如果相关）
if (issues.some(i => i.type === 'TASK_COMPLETED_NOT_ARCHIVED')) {
  await updateCurrentSprint()
  console.log('✅ 已更新 current-sprint.md')
}

// 4. 归档相关的设计文档（NEW）
for (const issue of issues) {
  if (issue.type === 'TASK_COMPLETED_NOT_ARCHIVED') {
    const sourcePath = issue.file
    const content = await readFile(sourcePath, 'utf-8')

    // 检查是否关联设计文档
    const designDocMatch = content.match(/\*\*相关设计\*\*:\s*(.+?)(?:\n|$)/) ||
                            content.match(/相关设计[:\s]+([^\n]+)/)

    if (designDocMatch) {
      let designDocPath = designDocMatch[1].trim()

      // 标准化路径
      if (!designDocPath.startsWith('docs/')) {
        designDocPath = `docs/${designDocPath}`
      }

      // 检查设计文档是否存在
      if (await fs.exists(designDocPath)) {
        // 使用与任务相同的归档目录
        const currentMonth = new Date().toISOString().slice(0, 7)
        const archiveDir = `docs/done/${currentMonth}`
        await fs.mkdir(archiveDir, { recursive: true })

        const designFileName = path.basename(designDocPath)
        const completedDesignPath = `${archiveDir}/${designFileName}`

        // 读取设计文档内容
        let designContent = await readFile(designDocPath, 'utf-8')

        // 添加完成标记
        const completionDate = new Date().toISOString().slice(0, 10)
        const taskId = extractTaskIdFromPath(sourcePath)

        if (!designContent.includes('**完成状态**')) {
          designContent = `---
**完成状态**: ✅ 已完成
**完成时间**: ${completionDate}
**关联任务**: ${taskId}
---

${designContent}`
        }

        // 写入到归档目录
        await writeFile(completedDesignPath, designContent, 'utf-8')

        // 删除原始设计文档
        await fs.unlink(designDocPath)

        console.log(`✅ 设计文档已归档: ${designFileName}`)
      }
    }
  }
}

console.log('\n✅ 自动同步完成')
```

### 步骤 5: 显示简要报告

```javascript
console.log('\n📊 同步摘要')
console.log(`- 修复问题: ${issues.length} 个`)

if (issues.length > 0) {
  console.log('\n提示: 定期运行 /sync-progress full 进行全面同步')
}
```

---

## 配置选项

在 `.claude/claude-task-pilot.local.md` 中配置：

```yaml
---
# 自动文档同步
auto_sync: true                # 启用自动同步（默认）
auto_sync_mode: "prompt"       # 同步模式
                               # - auto: 完全自动，不询问
                               # - prompt: 提示用户（默认）
                               # - silent: 静默修复，不显示输出
auto_sync_threshold: 3         # 问题数量阈值（超过此数量才提示）
auto_sync_exclude:             # 排除的文件/目录
  - "node_modules/"
  - ".git/"
---
```

---

## 同步模式说明

### 模式 1: auto（完全自动）

```yaml
auto_sync_mode: "auto"
```

**行为**:
- 检测到问题后自动修复
- 不询问用户
- 显示简要结果

**适用场景**:
- 信任自动同步逻辑
- 不想被打断
- 快速迭代开发

### 模式 2: prompt（提示用户，推荐）

```yaml
auto_sync_mode: "prompt"
```

**行为**:
- 检测到问题后询问用户
- 显示发现的问题
- 用户确认后才修复

**适用场景**:
- 需要控制同步时机
- 了解同步内容
- 大多数用户的推荐选择

### 模式 3: silent（静默修复）

```yaml
auto_sync_mode: "silent"
```

**行为**:
- 只修复明显的问题
- 不显示任何输出
- 用户无感知

**适用场景**:
- 不想看到同步信息
- 只修复确定的问题
- 高级用户

---

## 工作流程示例

### 场景 1: 编辑任务卡片，添加完成标记

```bash
# 1. 用户编辑任务卡片
vim docs/todo/backlog/task-001-feature.md

# 添加完成标记
**完成时间**: 2026-02-01

# 2. 保存文件，触发 Hook

# 3. Hook 自动检测
🔄 检测到文档不一致，自动同步中...

发现以下问题:
1. [TASK_COMPLETED_NOT_ARCHIVED] task-001-feature.md
   - 任务已完成但未归档

是否自动修复？ (Y/n) y

✅ 已归档: task-001-feature.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md

✅ 自动同步完成

📊 同步摘要
- 修复问题: 1 个
```

### 场景 2: 完全自动模式

```bash
# 1. 用户配置
auto_sync_mode: "auto"

# 2. 用户编辑任务卡片
vim docs/todo/backlog/task-001-feature.md
# 添加完成标记

# 3. 保存文件，触发 Hook

# 4. Hook 自动修复（不询问）
🔄 检测到文档不一致，自动同步中...

✅ 已归档: task-001-feature.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md

✅ 自动同步完成
```

### 场景 3: 禁用自动同步

```bash
# 1. 用户配置
auto_sync: false

# 2. 用户编辑任务卡片
vim docs/todo/backlog/task-001-feature.md
# 添加完成标记

# 3. Hook 不触发，用户需要手动同步
/sync-progress
```

---

## 与 SessionStart Hook 的配合

**SessionStart Hook** 也提供自动修复：

```markdown
## 🔍 检测到未记录的完成任务

发现以下任务已完成但未更新到项目文档：
- task-005: 实现用户登录
- task-006: 实现用户注册

正在自动更新文档...

✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 已更新 archive-index.md
```

**双层保障**:
1. **PostToolWrite Hook** - 操作后立即同步
2. **SessionStart Hook** - 启动时检查修复

---

## 性能优化

### 1. 轻量级检测

```javascript
// 不扫描所有文件，只检查必要的
const quickCheck = async () => {
  // 只检查刚操作的文件
  // 不扫描 backlog/ 和 done/ 目录
}
```

### 2. 智能触发

```javascript
// 不是所有文件操作都触发同步
// 只在检测到特定模式时触发
const shouldTrigger = (filePath) => {
  return filePath.match(/task-.*\.md$/) ||
         filePath.includes('session.md') ||
         filePath.includes('current-sprint.md')
}
```

### 3. 延迟执行

```javascript
// 避免频繁触发
let syncTimer = null

const scheduleSync = () => {
  if (syncTimer) {
    clearTimeout(syncTimer)
  }

  syncTimer = setTimeout(async () => {
    await performSync()
    syncTimer = null
  }, 1000)  // 1 秒后执行
}
```

---

## 错误处理

### E1. 同步失败

```javascript
try {
  await performSync()
} catch (error) {
  console.log('⚠️ 自动同步失败')
  console.log(`错误: ${error.message}`)
  console.log('\n建议: 手动运行 /sync-progress')
}
```

### E2. 文件冲突

```javascript
if (hasConflict) {
  console.log('⚠️ 检测到文件冲突')
  console.log('建议: 手动解决冲突后运行 /sync-progress')
  return null
}
```

---

## 最佳实践

### 1. 推荐配置（大多数用户）

```yaml
auto_sync: true
auto_sync_mode: "prompt"
```

### 2. 完全自动（高级用户）

```yaml
auto_sync: true
auto_sync_mode: "auto"
```

### 3. 手动控制

```yaml
auto_sync: false
# 定期手动运行 /sync-progress
```

---

## 与命令的配合

### 自动同步 + 手动命令

| 操作 | 自动同步 | 手动命令 |
|------|---------|---------|
| 编辑任务卡片 | ✅ Hook 自动触发 | /sync-progress |
| 完成任务 | ✅ 自动归档 | /complete-task |
| 全面同步 | ❌ 不触发 | /sync-progress full |
| 验证数据 | ❌ 不触发 | /sync-progress verify |

**推荐**:
- 日常使用 → 依赖自动同步
- 定期总结 → 手动运行 `/sync-progress full`
- 怀疑问题 → 手动运行 `/sync-progress verify`

---

## 调试

### 启用调试日志

```yaml
debug: true
```

### 查看同步历史

```bash
# 查看最近的同步操作
cat docs/session.md | grep "同步时间"
```

---

**版本**: 1.0.0
**最后更新**: 2026-02-01
