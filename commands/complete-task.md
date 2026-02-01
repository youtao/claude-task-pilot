---
description: 标记任务完成并自动更新相关文档
argument-hint: 任务ID或文件路径（可选，默认使用session.md中的当前任务）
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - AskUserQuestion
---

# Complete Task - 任务完成命令

## 功能描述

标记任务完成并自动更新所有相关文档，包括：
- 移动任务卡片到归档目录
- 更新 session.md
- 更新 current-sprint.md
- 更新 archive-index.md
- 智能推荐下一个任务

---

## 参数说明

**$ARGUMENTS**: 任务ID 或文件路径（可选）

### 格式 1: 任务ID
```bash
/complete-task task-001
/complete-task task-003
```

### 格式 2: 文件路径
```bash
/complete-task docs/todo/backlog/task-001-feature.md
/complete-task task-001-feature.md
```

### 格式 3: 不提供参数（默认）
```bash
/complete-task
```
自动从 session.md 读取"当前任务"

---

## 执行流程

### 步骤 1: 确定任务

```javascript
let taskPath = null

// 如果未提供参数
if (!ARGUMENTS || ARGUMENTS.trim() === '') {
  // 从 session.md 读取当前任务
  const sessionContent = await readFile('docs/session.md', 'utf-8')
  const match = sessionContent.match(/\[([^\]]+task-\d{3}-[\w-]+\.md)\]/)

  if (!match) {
    return {
      success: false,
      error: '未找到当前任务',
      message: '请提供任务ID或文件路径，例如: /complete-task task-001'
    }
  }

  taskPath = match[1]
  console.log(`从 session.md 读取当前任务: ${taskPath}`)
} else {
  // 使用提供的参数
  const input = ARGUMENTS.trim()

  // 判断是任务ID还是文件路径
  if (input.match(/^task-\d{3}$/)) {
    // 任务ID: task-001
    const taskId = input
    // 查找对应的任务文件
    const files = await glob(`docs/todo/backlog/${taskId}-*.md`)

    if (files.length === 0) {
      return {
        success: false,
        error: '未找到任务文件',
        taskId,
        message: `期望路径: docs/todo/backlog/${taskId}-<description>.md`
      }
    }

    taskPath = files[0]
  } else if (input.match(/^task-\d{3}-[\w-]+\.md$/)) {
    // 文件名: task-001-feature.md
    taskPath = `docs/todo/backlog/${input}`
  } else if (input.endsWith('.md')) {
    // 完整路径或相对路径
    taskPath = input.startsWith('docs/') ? input : `docs/${input}`
  } else {
    return {
      success: false,
      error: '参数格式不正确',
      input,
      message: '支持格式: task-001, task-001-feature.md, docs/todo/backlog/task-001-feature.md'
    }
  }
}
```

### 步骤 2: 验证任务文件

```javascript
// 检查文件是否存在
try {
  await fs.access(taskPath)
} catch {
  return {
    success: false,
    error: '任务文件不存在',
    path: taskPath,
    message: '请检查任务是否已完成或路径是否正确'
  }
}

// 检查文件路径
if (!taskPath.match(/docs\/todo\/backlog\/task-\d{3}-[\w-]+\.md$/)) {
  return {
    success: false,
    error: '文件路径不正确',
    path: taskPath,
    message: '任务文件应该在 docs/todo/backlog/ 目录下'
  }
}
```

### 步骤 3: 读取任务信息

```javascript
const content = await readFile(taskPath, 'utf-8')

// 提取任务信息
const taskId = content.match(/^# (task-\d{3):/)?.[1] ||
               path.basename(taskPath).match(/^(task-\d{3})/)?.[1]

const taskName = content.match(/^# task-\d{3}: (.+)/)?.[1] ||
                  path.basename(taskPath).replace('.md', '')

const priority = content.match(/\*\*优先级\*\*:\s*(P[0-2])/)?.[1] || 'P1'
const module = content.match(/\*\*模块\*\*:\s*(\w+)/)?.[1] || '未分类'
const milestone = content.match(/\*\*里程碑\*\*:\s*(.+)/)?.[1] || ''

// 读取创建时间
const createdDate = content.match(/\*\*创建时间\*\*:\s*(\d{4}-\d{2}-\d{2})/)?.[1] || new Date().toISOString().slice(0, 10)

console.log(`\n📋 任务信息`)
console.log(`ID: ${taskId}`)
console.log(`名称: ${taskName}`)
console.log(`优先级: ${priority}`)
console.log(`模块: ${module}`)
console.log(`创建时间: ${createdDate}`)
```

### 步骤 4: 确认完成

```javascript
console.log('\n准备标记任务为完成...')
console.log('将执行以下操作:')
console.log('1. 移动任务卡片到归档目录')
console.log('2. 创建完成报告')
console.log('3. 更新 session.md')
console.log('4. 更新 current-sprint.md')
console.log('5. 更新 archive-index.md')
console.log('6. 推荐下一个任务\n')

const confirm = await askUser('确认完成此任务？ (Y/n)', { default: true })

if (!confirm) {
  return { success: false, error: '用户取消操作' }
}
```

### 步骤 5: 准备归档路径

```javascript
// 确定归档目录（使用创建时间的月份）
const currentMonth = new Date().toISOString().slice(0, 7)  // "2026-02"
const archiveDir = `docs/done/${currentMonth}`

// 确保归档目录存在
try {
  await fs.mkdir(archiveDir, { recursive: true })
} catch (error) {
  // 目录可能已存在，忽略错误
}

// 生成归档文件名
const fileName = path.basename(taskPath)
const archivePath = `${archiveDir}/${fileName}`

// 检查文件是否已存在
if (await fs.exists(archivePath)) {
  // 添加时间戳
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-').slice(0, 19)
  const archivePath = `${archiveDir}/${fileName.replace('.md', '')}-${timestamp}.md`
}
```

### 步骤 6: 移动任务文件

```javascript
// 使用 Write 工具移动文件（触发 PostToolWrite Hook）
const fileContent = await readFile(taskPath, 'utf-8')

// 添加完成时间
const completionDate = new Date().toISOString().slice(0, 10)
const completionTime = new Date().toLocaleString('zh-CN')

let updatedContent = fileContent

// 如果还没有完成时间，添加它
if (!updatedContent.includes('**完成时间**')) {
  // 在创建时间后添加完成时间
  updatedContent = updatedContent.replace(
    /(\*\*创建时间\*\*:\s*\d{4}-\d{2}-\d{2})/,
    `$1\n**完成时间**: ${completionDate}`
  )
}

// 写入归档文件
await writeFile(archivePath, updatedContent, 'utf-8')

// 删除原文件
await fs.unlink(taskPath)

console.log(`✅ 任务已归档: ${archivePath}`)
```

### 步骤 6.5: 归档设计文档（NEW）

```javascript
// 检查任务卡片是否关联设计文档
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
    console.log(`📋 发现关联的设计文档: ${designDocPath}`)

    // 使用与任务相同的归档目录
    const currentMonth = new Date().toISOString().slice(0, 7)
    const archiveDir = `docs/done/${currentMonth}`
    await fs.mkdir(archiveDir, { recursive: true })

    // 提取设计文档文件名
    const designFileName = path.basename(designDocPath)
    const completedDesignPath = `${archiveDir}/${designFileName}`

    // 读取设计文档内容
    let designContent = await readFile(designDocPath, 'utf-8')

    // 添加完成标记
    const completionDate = new Date().toISOString().slice(0, 10)
    const completionTime = new Date().toLocaleString('zh-CN')

    if (!designContent.includes('**完成状态**')) {
      // 在文档开头添加完成状态
      designContent = `---
**完成状态**: ✅ 已完成
**完成时间**: ${completionDate}
**关联任务**: ${taskId}
**完成时间戳**: ${completionTime}
---

${designContent}`
    }

    // 写入到归档目录
    await writeFile(completedDesignPath, designContent, 'utf-8')

    // 删除原始设计文档
    await fs.unlink(designDocPath)

    console.log(`✅ 设计文档已归档: ${completedDesignPath}`)
  } else {
    console.log(`⚠️ 设计文档不存在: ${designDocPath}`)
  }
} else {
  console.log(`ℹ️ 任务未关联设计文档`)
}
```

### 步骤 7: 更新文档

#### 7.1 更新 session.md

```javascript
const sessionPath = 'docs/session.md'
let sessionContent = await readFile(sessionPath, 'utf-8')

// 清空"当前任务"部分
sessionContent = sessionContent.replace(
  /## 🎯 当前任务\n+[\s\S]*?(?=\n## |\n---|$)/,
  '## 🎯 当前任务\n\n暂无'
)

// 更新"上一个任务"部分
const previousTaskEntry = `| ${taskId} | ${taskName} | ${completionDate} | ${archivePath} |`

const previousTasksPattern = /## ⏳ 上一个任务\n+/
const previousTasksHeader = '## ⏳ 上一个任务\n\n| 任务ID | 描述 | 完成时间 | 归档位置 |\n|--------|------|----------|----------|\n'

if (sessionContent.includes('## ⏳ 上一个任务')) {
  // 已存在表格，添加新行
  sessionContent = sessionContent.replace(
    /(\|--------\|[\s\S]*?)(\n## |\n---|$)/,
    `$1${previousTaskEntry}$2`
  )
} else {
  // 不存在表格，创建新表格
  sessionContent = sessionContent.replace(
    /## 🎯 当前任务/,
    `${previousTasksHeader}${previousTaskEntry}\n\n## 🎯 当前任务`
  )
}

// 更新"更新时间"
sessionContent = sessionContent.replace(
  /\*\*更新时间\*\*:\s*\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}/,
  `**更新时间**: ${completionTime}`
)

await writeFile(sessionPath, sessionContent, 'utf-8')
console.log('✅ 已更新 session.md')
```

#### 7.2 更新 current-sprint.md

```javascript
const sprintPath = 'docs/todo/current-sprint.md'
let sprintContent = await readFile(sprintPath, 'utf-8')

// 更新任务状态从 🔄/⏳ 改为 ✅
sprintContent = sprintContent.replace(
  new RegExp(`\\|\\s*${taskId}\\s*\\|[^\\n]+\\|\\s*[🔄⏳]\\s*`, 'g'),
  (match) => {
    return match.replace(/[🔄⏳]/, '✅').replace(/进行中|待开始/, '完成')
  }
)

await writeFile(sprintPath, sprintContent, 'utf-8')
console.log('✅ 已更新 current-sprint.md')
```

#### 7.3 更新 archive-index.md

```javascript
const indexPath = 'docs/done/archive-index.md'
let indexContent = await readFile(indexPath, 'utf-8')

// 提取任务描述
const description = content.match(/##\s*(.+)/)?.[1] || taskName

// 添加归档记录
const archiveEntry = `| ${completionDate} | ${taskId}: ${taskName} | ${description} | ${fileName} |`

// 检查是否有当前月份的表格
const monthPattern = new RegExp(`###\\s+${currentMonth}`)

if (monthPattern.test(indexContent)) {
  // 找到月份表格，添加新行
  const tablePattern = new RegExp(`(###\\s+${currentMonth}[\\s\\S]*?\\|--------[\\s\\S]*?)(\\n###|\\n---|$)`)

  indexContent = indexContent.replace(
    tablePattern,
    `$1${archiveEntry}$2`
  )
} else {
  // 没有找到月份表格，创建新表格
  const newMonthTable = `
### ${currentMonth}

| 完成日期 | 任务 | 描述 | 归档文件 |
|---------|------|------|---------|
${archiveEntry}
`

  // 在文件开头或第一个月份表格之前插入
  const firstMonthMatch = indexContent.match(/\n### \d{4}-\d{2}/)

  if (firstMonthMatch) {
    // 在第一个月份表格之前插入
    indexContent = indexContent.replace(
      /(\n### \d{4}-\d{2})/,
      `${newMonthTable}$1`
    )
  } else {
    // 在文件开头插入
    indexContent = `${newMonthTable}\n${indexContent}`
  }
}

await writeFile(indexPath, indexContent, 'utf-8')
console.log('✅ 已更新 archive-index.md')
```

### 步骤 8: 推荐下一个任务

```javascript
// 调用 task-suggester agent
console.log('\n正在分析推荐下一个任务...\n')

const result = await runAgent('task-suggester', {
  context: {
    completedTasks: [taskId],
    currentProject: {
      name: path.basename(process.cwd()),
      docsRoot: 'docs'
    }
  }
})

return {
  success: true,
  taskId,
  taskName,
  archivePath,
  completionDate,
  nextTask: result.recommendations
}
```

---

## 使用示例

### 示例 1: 使用当前任务（默认）

```bash
/complete-task
```

**输出**:
```markdown
从 session.md 读取当前任务: docs/todo/backlog/task-001-feature.md

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

### 示例 2: 使用任务ID

```bash
/complete-task task-003
```

### 示例 3: 使用文件路径

```bash
/complete-task task-003-api-design.md
/complete-task docs/todo/backlog/task-003-api-design.md
```

---

## 错误处理

### E1. 未找到当前任务

```markdown
❌ 未找到当前任务

session.md 中没有记录当前任务。

请提供任务ID或文件路径，例如:
- /complete-task task-001
- /complete-task task-001-feature.md
```

### E2. 任务文件不存在

```markdown
❌ 任务文件不存在

路径: docs/todo/backlog/task-001-unknown.md

可能的原因:
1. 任务已归档
2. 任务文件路径不正确
3. 任务文件已被删除

建议:
1. 检查 docs/done/ 目录
2. 检查 docs/todo/current-sprint.md 确认任务状态
```

### E3. 归档目录创建失败

```markdown
❌ 无法创建归档目录

目录: docs/done/2026-02/

错误信息: {{error_message}}

建议:
1. 检查文件系统权限
2. 手动创建目录: mkdir -p docs/done/2026-02/
```

### E4. 文档更新失败

```markdown
⚠️ 部分文档更新失败

任务已归档，但以下文档更新失败:
- docs/session.md: {{error}}
- docs/todo/current-sprint.md: {{error}}

建议手动更新这些文档。
```

---

## 与 Hook 的配合

本命令与 PostToolWrite Hook 完美配合：

1. **命令使用 Write 工具移动任务文件**
2. **触发 PostToolWrite Hook**
3. **Hook 再次确认文档更新**

这提供了双重保障，确保文档总是正确的。

---

## 最佳实践

1. **工作完成后立即运行**：保持文档与实际同步
2. **检查完成报告**：确认所有文档已正确更新
3. **查看推荐任务**：利用智能推荐功能
4. **定期归档**：避免 backlog/ 目录积累过多已完成任务

---

## 相关命令

- `/convert-design-to-tasks` - 将设计文档转换为任务卡片
- `/setup-task-pilot` - 初始化项目结构

---

**版本**: 1.0.0
**最后更新**: 2026-02-01
