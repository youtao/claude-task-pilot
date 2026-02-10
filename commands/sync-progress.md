---
description: 全面同步所有项目文档，支持完成任务和转换设计文档
argument-hint: "[quick|full|verify] [--complete-task <id>] [--convert-design [path]]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - AskUserQuestion
  - Task
---

# Sync Progress

## 功能

全面同步项目文档，确保数据一致性。支持任务完成、设计文档转换。

**核心功能**:
- 📊 **进度同步**: 保持所有文档数据一致
- ✅ **任务完成**: 标记任务完成并自动归档
- 🔄 **设计转换**: 将计划文档转换为任务卡片（Superpowers + OMC）

**同步文档**:
- `session.md` - Session 状态
- `current-sprint.md` - 冲刺进度
- `roadmap.md` - 路线图
- `backlog/` - 任务卡片
- `archive-index.md` - 归档索引

## 参数说明

| 模式 | 说明 |
|------|------|
| **quick**（默认） | 同步 session.md + current-sprint.md，检测并归档完成任务 |
| **full** | 执行所有 quick 操作 + 更新 roadmap.md + 重建 archive-index.md |
| **verify** | 只检查数据一致性，不修改文件 |

| 选项 | 说明 |
|------|------|
| `--complete-task <id>` | 标记任务完成并归档，推荐下一个任务 |
| `--convert-design [path]` | 转换计划文档为任务，支持 `docs/plans/*.md` (Superpowers) 和 `.omc/plans/*.md` (OMC) |

## 执行流程

### 1. 读取状态

```javascript
const docs = {
  session: read('docs/session.md'),
  currentSprint: read('docs/todo/current-sprint.md'),
  roadmap: read('docs/todo/roadmap.md'),
  archiveIndex: read('docs/done/archive-index.md'),
  tasks: glob('docs/todo/backlog/task-*.md')
}
```

### 2. 检测不一致

```javascript
const issues = {
  completedNotArchived: tasks.filter(t => t.completed && !archived),
  statusMismatch: compareSprintVsTasks(docs.currentSprint, tasks),
  missingArchive: checkArchiveIndex(docs.archiveIndex, tasks)
}
```

### 3. 执行同步

**quick/full**:
```javascript
if (mode === 'quick' || mode === 'full') {
  updateSession()
  updateCurrentSprint()
  if (mode === 'full') {
    archiveCompletedTasks()
    regenerateArchiveIndex()
    updateRoadmap()
  }
}
```

**verify**:
```javascript
if (mode === 'verify') {
  reportIssues(issues)
  suggestFixes(issues)
}
```

### 4. 完成任务（--complete-task）

```javascript
completeTask(taskId) {
  addCompletionTime(taskId)
  moveToArchive(taskId)
  updateSession()
  updateCurrentSprint()
  updateArchiveIndex()
  archiveRelatedDesign(taskId)
  return suggestNextTask()
}
```

### 5. 转换计划（--convert-design）

```javascript
convertDesign(path) {
  // 扫描并选择最新计划
  const plans = scanPlans(['docs/plans/*.md', '.omc/plans/*.md'])
  const plan = path || selectLatest(plans)

  // 识别类型并调用对应 agent
  const type = detectPlanType(plan)  // 'superpowers' | 'omc'
  const agent = type === 'omc' ? 'omc-plan-to-tasks' : 'design-to-tasks'

  // 生成任务
  const tasks = await Task(agent, { planPath: plan })
  writeTasks(tasks)
  updateCurrentSprint(tasks)
  archivePlan(plan)
}
```

## 使用示例

```bash
# 快速同步（日常）
/sync-progress

# 完全同步（每周）
/sync-progress full

# 完成任务
/sync-progress --complete-task task-001

# 转换设计文档
/sync-progress --convert-design

# 组合使用
/sync-progress --complete-task task-001 full
/sync-progress -d .omc/plans/plan.md
```
