---
description: 全面同步所有项目文档，确保数据一致性
argument-hint: 同步模式（可选：quick/full/verify）
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - AskUserQuestion
---

# Sync Progress - 文档同步命令

## 功能描述

全面同步所有项目文档，确保数据一致性。解决"手动保存进度时只更新部分文档"的问题。

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

### 步骤 6: 生成同步报告

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

## 与其他命令的配合

### `/complete-task` vs `/sync-progress`

| 特性 | /complete-task | /sync-progress |
|------|---------------|----------------|
| 用途 | 标记单个任务完成 | 全面同步所有文档 |
| 更新范围 | 部分文档 | 所有文档 |
| 归档任务 | 是 | 是（full 模式） |
| 检测问题 | 否 | 是 |
| 推荐任务 | 是 | 否 |

**推荐工作流程**：
```bash
# 1. 完成单个任务
/complete-task

# 2. 完成多个任务后，全面同步
/sync-progress full

# 3. 定期检查（每周）
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

- `/complete-task` - 标记任务完成
- `/convert-design-to-tasks` - 将设计文档转换为任务卡片
- `/setup-task-pilot` - 初始化项目结构

---

**版本**: 1.0.0
**最后更新**: 2026-02-01
