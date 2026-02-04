---
description: 全面同步所有项目文档，支持完成任务和转换设计文档
argument-hint: [quick|full|verify] [--complete-task <id>] [--convert-design [path]]
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - AskUserQuestion
  - Task
---

# Sync Progress - 文档同步命令

## 功能描述

全面同步所有项目文档，确保数据一致性。支持任务完成、设计文档转换和多种同步模式。

**核心功能**：
- 📊 **进度同步**：保持所有文档数据一致
- ✅ **任务完成**：标记任务完成并自动归档
- 🔄 **设计转换**：将设计文档转换为任务卡片

**同步的文档**：
- ✅ `session.md` - 当前 Session 状态
- ✅ `current-sprint.md` - 当前冲刺进度
- ✅ `roadmap.md` - 长期路线图进度
- ✅ `backlog/` - 任务卡片状态
- ✅ `archive-index.md` - 归档索引
- ✅ 检测未归档的完成任务
- ✅ 检测孤儿文件（backlog/ 中已完成但未归档的任务）

---

## 参数说明

**$ARGUMENTS**: 同步模式（可选）

### 模式 1: quick（快速同步，默认）

```bash
/sync-progress
/sync-progress quick
```

**行为**：
- 同步 session.md 和 current-sprint.md
- 检测并归档明显的完成任务
- 快速验证数据一致性

**适用场景**：
- 日常工作中的快速同步
- 完成 1-2 个任务后

### 模式 2: full（完全同步）

```bash
/sync-progress full
```

**行为**：
- 执行所有快速同步操作
- 更新 roadmap.md 进度
- 扫描所有任务卡片，更新状态
- 重新生成 archive-index.md（如果损坏）
- 详细的同步报告

**适用场景**：
- 阶段性总结（每周、每月）
- 长时间工作后
- 感觉文档不一致时

### 模式 3: verify（验证模式）

```bash
/sync-progress verify
```

**行为**：
- 不修改任何文件
- 只检查数据一致性
- 报告发现的问题
- 提供修复建议

**适用场景**：
- 怀疑文档有问题时
- 定期检查（每周）
- 同步前的预检查

### 选项 1: 完成任务

```bash
/sync-progress --complete-task [task-id]
/sync-progress -c [task-id]
```

**功能**：标记任务完成并归档。

**参数**：
- `task-id`：任务ID（如：task-001）或文件名（如：task-001-feature.md）
- 未提供参数：从 session.md 读取当前任务

**行为**：
- 添加完成时间到任务卡片
- 移动到归档目录（`docs/done/YYYY-MM/`）
- 更新 `session.md`（当前任务 → 上一个任务）
- 更新 `current-sprint.md`（状态 → ✅）
- 更新 `archive-index.md`（添加归档记录）
- 自动归档关联的设计文档（如果存在）
- 调用 task-suggester agent 推荐下一个任务

**适用场景**：
- 任务完成后立即归档
- 需要同步更新所有相关文档
- 希望获得下一个任务推荐

### 选项 2: 转换设计文档

```bash
/sync-progress --convert-design [design-doc-path]
/sync-progress -d [design-doc-path]
```

**功能**：将设计文档转换为任务卡片。

**参数**：
- `design-doc-path`：设计文档路径（可选）
- 未提供参数：自动使用最新的设计文档（`docs/plans/*-design.md`）

**行为**：
- 分析设计文档内容
- 生成任务卡片到 `docs/todo/backlog/`
- 更新 `current-sprint.md`
- 可与同步模式组合使用

**适用场景**：
- 完成 brainstorming 后生成任务
- 需要将设计文档转化为可执行任务
- 快速启动新功能开发

### 选项 3: 组合操作

```bash
# 转换设计文档后立即完全同步
/sync-progress --convert-design --full

# 完成任务后立即同步
/sync-progress --complete-task task-001 --full

# 转换设计文档并验证结果
/sync-progress -d docs/plans/design.md --verify
```

**说明**：选项可以与同步模式组合使用，实现一键完成多个操作。

### 选项 4: 帮助信息

```bash
/sync-progress --help
```

显示所有可用参数和使用示例。

---

## 执行流程

### 步骤 1: 读取项目状态

```javascript
// 读取关键文档
const sessionPath = 'docs/session.md'
const sprintPath = 'docs/todo/current-sprint.md'
const roadmapPath = 'docs/todo/roadmap.md'
const archiveIndexPath = 'docs/done/archive-index.md'

// 读取任务卡片
const backlogFiles = await glob('docs/todo/backlog/task-*.md')
const doneFiles = await glob('docs/done/**/task-*.md')

console.log(`📊 项目状态`)
console.log(`- backlog/: ${backlogFiles.length} 个任务`)
console.log(`- done/: ${doneFiles.length} 个已完成任务`)
console.log(`- 总计: ${backlogFiles.length + doneFiles.length} 个任务\n`)
```

### 步骤 2: 检测不一致

```javascript
const issues = []

// 检查 1: backlog 中已完成的任务（未归档）
for (const file of backlogFiles) {
  const content = await readFile(file, 'utf-8')

  // 检查是否有完成标记
  if (content.includes('**完成时间**') || content.includes('状态: ✅')) {
    issues.push({
      type: 'UNARCHIVED_TASK',
      file,
      message: '任务已完成但未归档'
    })
  }
}

// 检查 2: current-sprint.md 状态不一致
const sprintContent = await readFile(sprintPath, 'utf-8')
const sprintTasks = parseSprintTasks(sprintContent)

for (const task of sprintTasks) {
  const isInBacklog = backlogFiles.some(f => f.includes(task.id))
  const isInDone = doneFiles.some(f => f.includes(task.id))

  if (task.status === '✅ 完成' && isInBacklog) {
    issues.push({
      type: 'STATUS_MISMATCH',
      taskId: task.id,
      message: `current-sprint.md 标记为完成，但任务仍在 backlog/`
    })
  }

  if ((task.status === '🔄 进行中' || task.status === '⏳ 待开始') && isInDone) {
    issues.push({
      type: 'STATUS_MISMATCH',
      taskId: task.id,
      message: `current-sprint.md 标记为进行中/待开始，但任务已在 done/`
    })
  }
}

// 检查 3: archive-index.md 缺失记录
const archivedTasks = parseArchiveIndex(archiveIndexPath)

for (const file of doneFiles) {
  const taskId = extractTaskId(file)

  if (!archivedTasks.includes(taskId)) {
    issues.push({
      type: 'MISSING_INDEX',
      file,
      message: '任务已归档但未在 archive-index.md 中记录'
    })
  }
}

console.log(`🔍 检测到 ${issues.length} 个问题\n`)

if (mode === 'verify') {
  // 验证模式，只报告问题
  return {
    success: true,
    mode: 'verify',
    issues,
    summary: generateIssueReport(issues)
  }
}
```

### 步骤 3: 确认同步

```javascript
if (mode === 'full' || issues.length > 0) {
  console.log(`准备同步以下内容:`)

  if (mode === 'full') {
    console.log(`✓ 更新 session.md`)
    console.log(`✓ 更新 current-sprint.md`)
    console.log(`✓ 更新 roadmap.md`)
    console.log(`✓ 归档未归档的完成任务`)
    console.log(`✓ 更新 archive-index.md`)
  }

  if (issues.length > 0) {
    console.log(`\n修复以下问题:`)
    issues.forEach((issue, i) => {
      console.log(`${i + 1}. [${issue.type}] ${issue.message}`)
    })
  }

  const confirm = await askUser('\n确认执行同步？ (Y/n)', { default: true })
  if (!confirm) {
    return { success: false, error: '用户取消操作' }
  }
}
```

### 步骤 4: 快速同步（quick/full）

#### 4.1 更新 session.md

```javascript
// 提取当前任务
const currentTasks = sprintTasks.filter(t => t.status === '🔄 进行中')

// 提取最近完成的任务（最多 5 个）
const recentCompleted = doneFiles
  .sort((a, b) => {
    const statA = await fs.stat(a)
    const statB = await fs.stat(b)
    return statB.mtime - statA.mtime
  })
  .slice(0, 5)
  .map(file => {
    const taskId = extractTaskId(file)
    const content = await readFile(file, 'utf-8')
    const taskName = content.match(/^# (task-\d{3}: .+)/)?.[1] || taskId
    const completionDate = content.match(/\*\*完成时间\*\*:\s*(\d{4}-\d{2}-\d{2})/)?.[1] || '未知'

    return { taskId, taskName, completionDate, file }
  })

// 生成新的 session.md
const newSessionContent = generateSessionMd(currentTasks, recentCompleted)
await writeFile(sessionPath, newSessionContent, 'utf-8')

console.log('✅ 已更新 session.md')
```

#### 4.2 更新 current-sprint.md

```javascript
// 同步任务状态
let updatedSprintContent = sprintContent

// 标记在 backlog 中已完成的任务为 ✅
for (const file of backlogFiles) {
  const content = await readFile(file, 'utf-8')

  if (content.includes('**完成时间**')) {
    const taskId = extractTaskId(file)

    updatedSprintContent = updatedSprintContent.replace(
      new RegExp(`\\|\\s*${taskId}\\s*\\|[^\\n]+\\|\\s*[🔄⏳]\\s*`),
      (match) => match.replace(/[🔄⏳]/, '✅').replace(/进行中|待开始/, '完成')
    )
  }
}

await writeFile(sprintPath, updatedSprintContent, 'utf-8')

console.log('✅ 已更新 current-sprint.md')
```

### 步骤 5: 完全同步（full only）

#### 5.1 归档未归档的任务

```javascript
if (mode === 'full') {
  const currentMonth = new Date().toISOString().slice(0, 7)
  const archiveDir = `docs/done/${currentMonth}`

  await fs.mkdir(archiveDir, { recursive: true })

  for (const issue of issues) {
    if (issue.type === 'UNARCHIVED_TASK') {
      const file = issue.file
      const fileName = path.basename(file)
      const archivePath = `${archiveDir}/${fileName}`

      // 移动文件
      const content = await readFile(file, 'utf-8')
      await writeFile(archivePath, content, 'utf-8')
      await fs.unlink(file)

      console.log(`✅ 已归档: ${fileName}`)
    }
  }
}
```

#### 5.2 更新 archive-index.md

```javascript
if (mode === 'full') {
  // 重新生成 archive-index.md
  const allDoneFiles = await glob('docs/done/**/task-*.md')

  // 按月份分组
  const tasksByMonth = {}
  for (const file of allDoneFiles) {
    const match = file.match(/done\/(\d{4}-\d{2})\//)
    if (match) {
      const month = match[1]
      if (!tasksByMonth[month]) {
        tasksByMonth[month] = []
      }
      tasksByMonth[month].push(file)
    }
  }

  // 生成新的 archive-index.md
  const newIndexContent = generateArchiveIndex(tasksByMonth)
  await writeFile(archiveIndexPath, newIndexContent, 'utf-8')

  console.log('✅ 已更新 archive-index.md')
}
```

#### 5.3 更新 roadmap.md

```javascript
if (mode === 'full') {
  const roadmapContent = await readFile(roadmapPath, 'utf-8')

  // 计算进度
  const totalTasks = backlogFiles.length + doneFiles.length
  const completedTasks = doneFiles.length
  const progress = Math.round((completedTasks / totalTasks) * 100)

  // 更新进度条
  let updatedRoadmap = roadmapContent

  // 查找并更新进度标记
  updatedRoadmap = updatedRoadmap.replace(
    /进度:\s*\d+%/,
    `进度: ${progress}%`
  )

  await writeFile(roadmapPath, updatedRoadmap, 'utf-8')

  console.log('✅ 已更新 roadmap.md`)
}
```

### 步骤 6.5: 完成任务流程（当使用 --complete-task 时）

#### 6.5.1 确定任务

```javascript
let taskPath = null

// 如果未提供参数
if (!ARGUMENTS.includes('--complete-task') && !ARGUMENTS.includes('-c')) {
  // 跳过任务完成流程
  return
}

const taskArg = ARGUMENTS.match(/(?:--complete-task|-c)\s+(\S+)/)?.[1]

if (!taskArg) {
  // 从 session.md 读取当前任务
  const sessionContent = await readFile('docs/session.md', 'utf-8')
  const match = sessionContent.match(/\[([^\]]+task-\d{3}-[\w-]+\.md)\]/)

  if (!match) {
    console.log('⚠️ 未找到当前任务，请提供任务ID')
    return
  }

  taskPath = match[1]
  console.log(`从 session.md 读取当前任务: ${taskPath}`)
} else {
  // 使用提供的参数
  const input = taskArg.trim()

  if (input.match(/^task-\d{3}$/)) {
    // 任务ID: task-001
    const files = await glob(`docs/todo/backlog/${input}-*.md`)
    if (files.length === 0) {
      console.log(`❌ 未找到任务: ${input}`)
      return
    }
    taskPath = files[0]
  } else if (input.match(/^task-\d{3}-[\w-]+\.md$/)) {
    // 文件名: task-001-feature.md
    taskPath = `docs/todo/backlog/${input}`
  } else if (input.endsWith('.md')) {
    // 完整路径或相对路径
    taskPath = input.startsWith('docs/') ? input : `docs/${input}`
  }
}
```

#### 6.5.2 读取任务信息

```javascript
const content = await readFile(taskPath, 'utf-8')

// 提取任务信息
const taskId = content.match(/^# (task-\d{3}:)/)?.[1] ||
               path.basename(taskPath).match(/^(task-\d{3})/)?.[1]

const taskName = content.match(/^# task-\d{3}: (.+)/)?.[1] ||
                  path.basename(taskPath).replace('.md', '')

const priority = content.match(/\*\*优先级\*\*:\s*(P[0-2])/)?.[1] || 'P1'
const module = content.match(/\*\*模块\*\*:\s*(\w+)/)?.[1] || '未分类'

// 读取创建时间
const createdDate = content.match(/\*\*创建时间\*\*:\s*(\d{4}-\d{2}-\d{2})/)?.[1] ||
                     new Date().toISOString().slice(0, 10)

console.log(`\n📋 任务信息`)
console.log(`ID: ${taskId}`)
console.log(`名称: ${taskName}`)
console.log(`优先级: ${priority}`)
console.log(`模块: ${module}`)
```

#### 6.5.3 准备归档

```javascript
// 确定归档目录（使用创建时间的月份）
const currentMonth = new Date().toISOString().slice(0, 7)
const archiveDir = `docs/done/${currentMonth}`

await fs.mkdir(archiveDir, { recursive: true })

const fileName = path.basename(taskPath)
const archivePath = `${archiveDir}/${fileName}`

// 添加完成时间
const completionDate = new Date().toISOString().slice(0, 10)
const completionTime = new Date().toLocaleString('zh-CN')

let updatedContent = content

if (!updatedContent.includes('**完成时间**')) {
  updatedContent = updatedContent.replace(
    /(\*\*创建时间\*\*:\s*\d{4}-\d{2}-\d{2})/,
    `$1\n**完成时间**: ${completionDate}`
  )
}
```

#### 6.5.4 归档任务和设计文档

```javascript
// 写入归档文件
await writeFile(archivePath, updatedContent, 'utf-8')

// 删除原文件
await fs.unlink(taskPath)

console.log(`✅ 任务已归档: ${archivePath}`)

// 检查并归档关联的设计文档
const designDocMatch = content.match(/\*\*相关设计\*\*:\s*(.+?)(?:\n|$)/) ||
                        content.match(/相关设计[:\s]+([^\n]+)/)

if (designDocMatch) {
  let designDocPath = designDocMatch[1].trim()

  if (!designDocPath.startsWith('docs/')) {
    designDocPath = `docs/${designDocPath}`
  }

  if (await fs.exists(designDocPath)) {
    console.log(`📋 发现关联的设计文档: ${designDocPath}`)

    const designFileName = path.basename(designDocPath)
    const completedDesignPath = `${archiveDir}/${designFileName}`

    let designContent = await readFile(designDocPath, 'utf-8')

    if (!designContent.includes('**完成状态**')) {
      designContent = `---
**完成状态**: ✅ 已完成
**完成时间**: ${completionDate}
**关联任务**: ${taskId}
**完成时间戳**: ${completionTime}
---

${designContent}`
    }

    await writeFile(completedDesignPath, designContent, 'utf-8')
    await fs.unlink(designDocPath)

    console.log(`✅ 设计文档已归档: ${completedDesignPath}`)
  }
}
```

#### 6.5.5 更新文档

```javascript
// 更新 session.md
const sessionPath = 'docs/session.md'
let sessionContent = await readFile(sessionPath, 'utf-8')

sessionContent = sessionContent.replace(
  /## 🎯 当前任务\n+[\s\S]*?(?=\n## |\n---|$)/,
  '## 🎯 当前任务\n\n暂无'
)

const previousTaskEntry = `| ${taskId} | ${taskName} | ${completionDate} | ${archivePath} |`

if (sessionContent.includes('## ⏳ 上一个任务')) {
  sessionContent = sessionContent.replace(
    /(\|--------\|[\s\S]*?)(\n## |\n---|$)/,
    `$1${previousTaskEntry}$2`
  )
} else {
  const previousTasksHeader = '## ⏳ 上一个任务\n\n| 任务ID | 描述 | 完成时间 | 归档位置 |\n|--------|------|----------|----------|\n'
  sessionContent = sessionContent.replace(
    /## 🎯 当前任务/,
    `${previousTasksHeader}${previousTaskEntry}\n\n## 🎯 当前任务`
  )
}

await writeFile(sessionPath, sessionContent, 'utf-8')
console.log('✅ 已更新 session.md')

// 更新 current-sprint.md
const sprintPath = 'docs/todo/current-sprint.md'
let sprintContent = await readFile(sprintPath, 'utf-8')

sprintContent = sprintContent.replace(
  new RegExp(`\\|\\s*${taskId}\\s*\\|[^\\n]+\\|\\s*[🔄⏳]\\s*`, 'g'),
  (match) => match.replace(/[🔄⏳]/, '✅').replace(/进行中|待开始/, '完成')
)

await writeFile(sprintPath, sprintContent, 'utf-8')
console.log('✅ 已更新 current-sprint.md')

// 更新 archive-index.md
const indexPath = 'docs/done/archive-index.md'
let indexContent = await readFile(indexPath, 'utf-8')

const archiveEntry = `| ${completionDate} | ${taskId}: ${taskName} | ${taskName} | ${fileName} |`

const monthPattern = new RegExp(`###\\s+${currentMonth}`)

if (monthPattern.test(indexContent)) {
  const tablePattern = new RegExp(`(###\\s+${currentMonth}[\\s\\S]*?\\|--------[\\s\\S]*?)(\\n###|\\n---|$)`)
  indexContent = indexContent.replace(tablePattern, `$1${archiveEntry}$2`)
} else {
  const newMonthTable = `
### ${currentMonth}

| 完成日期 | 任务 | 描述 | 归档文件 |
|---------|------|------|---------|
${archiveEntry}
`

  const firstMonthMatch = indexContent.match(/\n### \d{4}-\d{2}/)

  if (firstMonthMatch) {
    indexContent = indexContent.replace(/(\n### \d{4}-\d{2})/, `${newMonthTable}$1`)
  } else {
    indexContent = `${newMonthTable}\n${indexContent}`
  }
}

await writeFile(indexPath, indexContent, 'utf-8')
console.log('✅ 已更新 archive-index.md')
```

#### 6.5.6 推荐下一个任务

```javascript
console.log('\n正在分析推荐下一个任务...\n')

// 调用 task-suggester agent
const result = await runAgent('task-suggester', {
  context: {
    completedTasks: [taskId],
    currentProject: {
      name: path.basename(process.cwd()),
      docsRoot: 'docs'
    }
  }
})

console.log('✅ 任务完成流程结束\n')
```

### 步骤 6.6: 转换设计文档流程（当使用 --convert-design 时）

#### 6.6.1 确定设计文档

```javascript
if (!ARGUMENTS.includes('--convert-design') && !ARGUMENTS.includes('-d')) {
  // 跳过设计文档转换流程
  return
}

let designDoc = null
const designArg = ARGUMENTS.match(/(?:--convert-design|-d)(?:\s+(\S+))?/)?.[1]

if (!designArg) {
  // 自动查找最新设计文档
  const designDocs = await glob('docs/plans/*-design.md')

  if (designDocs.length === 0) {
    console.log('❌ 未找到设计文档')
    console.log('期望路径: docs/plans/YYYY-MM-DD-<topic>-design.md')
    return
  }

  designDocs.sort((a, b) => {
    const statA = await fs.stat(a)
    const statB = await fs.stat(b)
    return statB.mtime - statA.mtime
  })

  designDoc = designDocs[0]
  console.log(`使用最新设计文档: ${designDoc}`)
} else {
  // 使用提供的路径
  designDoc = designArg

  if (!designDoc.startsWith('/') && !designDoc.match(/^[A-Z]:/i)) {
    designDoc = path.join(process.cwd(), designDoc)
  }
}
```

#### 6.6.2 验证设计文档

```javascript
// 检查文件是否存在
try {
  await fs.access(designDoc)
} catch {
  console.log(`❌ 设计文档不存在: ${designDoc}`)
  return
}

// 检查文件格式
if (!designDoc.endsWith('.md')) {
  console.log('❌ 文件格式不支持，设计文档必须是 Markdown 格式（.md）')
  return
}

// 读取并验证内容
const content = await readFile(designDoc, 'utf-8')

if (!content || content.trim().length < 50) {
  console.log('❌ 设计文档内容为空或过于简单')
  console.log('请完善设计文档内容，至少包含功能描述和技术方案')
  return
}

const basename = path.basename(designDoc)
console.log(`\n📋 分析设计文档: ${basename}`)
console.log(`路径: ${designDoc}`)
console.log(`大小: ${content.length} 字符\n`)
```

#### 6.6.3 调用 Design-to-Tasks Agent

```javascript
console.log('准备将设计文档转换为任务卡片...')
console.log('任务将创建到: docs/todo/backlog/\n')

const confirm = await askUser('是否继续？ (Y/n)', { default: true })

if (!confirm) {
  console.log('❌ 用户取消操作')
  return
}

// 调用 agent 执行转换
const result = await runAgent('design-to-tasks', {
  designDoc: designDoc,
  mode: 'manual',
  verbose: true
})

if (result.success) {
  console.log('\n' + '='.repeat(60))
  console.log('✅ 转换完成！')
  console.log('='.repeat(60))

  console.log(`\n生成任务: ${result.taskCount} 个`)
  console.log(`任务ID: ${result.taskIds.join(', ')}`)

  console.log('\n下一步:')
  console.log(`1. 查看任务列表: cat docs/todo/current-sprint.md`)
  console.log(`2. 开始任务: cat docs/todo/backlog/${result.firstTask}.md`)
} else {
  console.log('\n❌ 转换失败')
  console.log(`错误: ${result.error}`)
}

console.log('✅ 设计文档转换流程结束\n')
```

### 步骤 7: 生成同步报告

```javascript
console.log('\n' + '='.repeat(60))
console.log('✅ 同步完成！')
console.log('='.repeat(60))

console.log(`\n📊 同步摘要`)

if (issues.length > 0) {
  console.log(`修复问题: ${issues.length} 个`)
  console.log(`- 归档未归档任务: ${issues.filter(i => i.type === 'UNARCHIVED_TASK').length} 个`)
  console.log(`- 修复状态不一致: ${issues.filter(i => i.type === 'STATUS_MISMATCH').length} 个`)
  console.log(`- 补充归档索引: ${issues.filter(i => i.type === 'MISSING_INDEX').length} 个`)
}

console.log(`\n当前状态:`)
console.log(`- backlog/: ${backlogFiles.length} 个任务`)
console.log(`- done/: ${doneFiles.length} 个已完成任务`)

if (mode === 'full') {
  const totalTasks = backlogFiles.length + doneFiles.length
  const progress = Math.round((doneFiles.length / totalTasks) * 100)
  console.log(`- 总进度: ${progress}%`)
}

console.log(`\n下一步:`)
console.log(`1. 查看当前任务: cat docs/session.md`)
console.log(`2. 查看冲刺进度: cat docs/todo/current-sprint.md`)

if (backlogFiles.length > 0) {
  console.log(`3. 继续任务: cat docs/todo/backlog/${extractTaskId(backlogFiles[0])}.md`)
} else {
  console.log(`3. 所有任务已完成！🎉`)
}

return {
  success: true,
  mode,
  issuesFixed: issues.length,
  backlogCount: backlogFiles.length,
  doneCount: doneFiles.length
}
```

---

## 使用示例

### 示例 1: 快速同步（默认）

```bash
/sync-progress
```

**输出**:
```markdown
📊 项目状态
- backlog/: 5 个任务
- done/: 10 个已完成任务
- 总计: 15 个任务

🔍 检测到 2 个问题

准备同步以下内容:
✓ 更新 session.md
✓ 更新 current-sprint.md
✓ 归档未归档的完成任务

修复以下问题:
1. [UNARCHIVED_TASK] 任务已完成但未归档
2. [STATUS_MISMATCH] current-sprint.md 标记为完成，但任务仍在 backlog/

确认执行同步？ (Y/n) y

✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 已归档: task-005-feature.md

============================================================
✅ 同步完成！
============================================================

📊 同步摘要
修复问题: 2 个
- 归档未归档任务: 1 个
- 修复状态不一致: 1 个

当前状态:
- backlog/: 4 个任务
- done/: 11 个已完成任务

下一步:
1. 查看当前任务: cat docs/session.md
2. 查看冲刺进度: cat docs/todo/current-sprint.md
3. 继续任务: cat docs/todo/backlog/task-006.md
```

### 示例 2: 完全同步

```bash
/sync-progress full
```

**输出**:
```markdown
📊 项目状态
- backlog/: 8 个任务
- done/: 22 个已完成任务
- 总计: 30 个任务

🔍 检测到 5 个问题

准备同步以下内容:
✓ 更新 session.md
✓ 更新 current-sprint.md
✓ 更新 roadmap.md
✓ 归档未归档的完成任务
✓ 更新 archive-index.md

修复以下问题:
1. [UNARCHIVED_TASK] 任务已完成但未归档
...

确认执行同步？ (Y/n) y

[执行所有同步操作...]

✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 已更新 roadmap.md
✅ 已更新 archive-index.md
✅ 总进度: 73%
```

### 示例 3: 验证模式

```bash
/sync-progress verify
```

**输出**:
```markdown
📊 项目状态
- backlog/: 5 个任务
- done/: 10 个已完成任务

🔍 检测到 1 个问题

验证模式 - 不修改任何文件

发现问题:
1. [UNARCHIVED_TASK] task-008-feature.md
   - 任务已完成但未归档
   - 建议: 运行 /sync-progress 归档此任务

结论: 数据基本一致，发现 1 个小问题
建议: 运行 /sync-progress 修复
```

### 示例 4: 完成任务

```bash
/sync-progress --complete-task task-001
```

**输出**:
```markdown
📋 任务信息
ID: task-001
名称: 实现用户登录功能
优先级: P0
模块: backend
创建时间: 2026-02-01

准备标记任务为完成...
将执行以下操作:
1. 移动任务卡片到归档目录
2. 创建完成报告
3. 更新 session.md
4. 更新 current-sprint.md
5. 更新 archive-index.md
6. 推荐下一个任务

确认完成此任务？ (Y/n) y

✅ 任务已归档: docs/done/2026-02/task-001-feature.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 已更新 archive-index.md

正在分析推荐下一个任务...

[task-suggester 输出推荐...]
```

### 示例 5: 转换设计文档

```bash
/sync-progress --convert-design
```

**输出**:
```markdown
查找最新设计文档...

找到: docs/plans/2026-02-01-user-authentication-design.md
修改时间: 2026-02-01 10:30:00

📋 分析设计文档: 2026-02-01-user-authentication-design.md
路径: docs/plans/2026-02-01-user-authentication-design.md
大小: 3520 字符

准备将设计文档转换为任务卡片...
任务将创建到: docs/todo/backlog/

是否继续？ (Y/n) y

[调用 agent 转换...]

============================================================
✅ 转换完成！
============================================================

生成任务: 5 个
任务ID: task-001, task-002, task-003, task-004, task-005

下一步:
1. 查看任务列表: cat docs/todo/current-sprint.md
2. 开始任务: cat docs/todo/backlog/task-001-design-user-data-model.md
```

### 示例 6: 组合操作

```bash
/sync-progress --complete-task task-001 --full
```

**输出**:
```markdown
📋 任务信息
ID: task-001
...

✅ 任务已归档: docs/done/2026-02/task-001-feature.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 已更新 archive-index.md

📊 开始完全同步...

✅ 已更新 roadmap.md
✅ 总进度: 75%
```

---

## 错误处理

### E1. 文件不存在

```markdown
❌ 文件不存在

路径: docs/session.md

可能的原因:
1. 项目未初始化
2. 文件被误删

建议:
1. 运行 /setup-task-pilot 重新初始化
2. 或从备份恢复
```

### E2. 权限不足

```markdown
❌ 权限不足

无法写入文件: docs/session.md

建议:
1. 检查文件权限
2. 关闭可能打开文件的编辑器
3. 尝试以适当权限运行
```

### E3. 数据冲突

```markdown
⚠️ 数据冲突

检测到多个冲突的状态:
- task-005: backlog/ 和 done/ 都存在

请手动解决:
1. 确定正确的任务位置
2. 删除重复文件
3. 重新运行 /sync-progress
```

---

## 相关命令

### `/setup-task-pilot`
初始化项目结构，创建必要的文档目录。

**用法**：
```bash
/setup-task-pilot
```

**适用场景**：
- 新项目首次使用
- 文档结构损坏时重建

---

## 推荐工作流程

```bash
# 1. 初始化项目（首次使用）
/setup-task-pilot

# 2. 转换设计文档为新任务
/sync-progress --convert-design

# 3. 开始执行任务...
# （工作过程中）

# 4. 完成任务
/sync-progress --complete-task task-001

# 5. 定期同步进度
/sync-progress

# 6. 每周完全同步
/sync-progress full

# 7. 定期验证数据一致性
/sync-progress verify
```

---

## 最佳实践

### 1. 定期同步

```bash
# 每天: 快速同步
/sync-progress

# 每周: 完全同步
/sync-progress full

# 每月: 验证数据一致性
/sync-progress verify
```

### 2. 阶段性总结

```bash
# 完成 Sprint 或 Milestone 后
/sync-progress full

# 查看最终报告
cat docs/session.md
cat docs/todo/current-sprint.md
cat docs/todo/roadmap.md
```

### 3. 怀疑问题时

```bash
# 先验证，不修改文件
/sync-progress verify

# 确认有问题后，再修复
/sync-progress full
```

---

## 自动化建议

### Git 钩子

在 `.git/hooks/pre-commit` 中添加：

```bash
#!/bin/bash
# 提交前自动同步文档
claude-code /sync-progress quick
```

### 定时任务

使用 cron 或 launchd 定期运行：

```bash
# 每天下午 6 点同步
0 18 * * * cd ~/project && claude-code /sync-progress
```

---

## 相关命令

- `/setup-task-pilot` - 初始化项目结构

---

**版本**: 2.0.0
**最后更新**: 2026-02-04
