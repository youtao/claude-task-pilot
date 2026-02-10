---
name: omc-plan-to-tasks
description: 将 OMC 计划文件（.omc/plans/*.md）转换为任务卡片。OMC 计划包含结构化的 Detailed TODOs 章节。
model: sonnet
color: "8B5CF6"
tools:
  - Read
  - Write
  - Glob
---

# OMC Plan-to-Tasks Agent

## 功能

解析 OMC 计划文件（`.omc/plans/*.md`），将 Detailed TODOs 转换为任务卡片。

## 触发

- **手动**: `--convert-design .omc/plans/xxx.md`
- **自动**: `--convert-design` 检测到 OMC 计划文件

## 流程

### 1. 读取并验证

```javascript
const planPath = ARGUMENTS || findLatest('.omc/plans/*.md')
const content = readFile(planPath)

// 验证 OMC 格式
const isOmcPlan = content.includes('## Detailed TODOs') ||
                  content.match(/\*\*Created\*\*:/)
if (!isOmcPlan) {
  return error("不是 OMC 计划文件，缺少 ## Detailed TODOs")
}
```

### 2. 提取元数据

```javascript
const metadata = {
  title: content.match(/^# (.+)$/m)?.[1],
  created: content.match(/\*\*Created\*\*:\s*(.+)/)?.[1],
  status: content.match(/\*\*Status\*\*:\s*(.+)/)?.[1],
  context: extractSection(content, '## Context'),
  objectives: extractSection(content, '## Work Objectives')
}
```

### 3. 解析 Detailed TODOs

```javascript
const todosSection = extractSection(content, '## Detailed TODOs')
const stepPattern = /### Step\s+(\d+):\s*(.+?)(?=\n###|\n##|$)/gs
const steps = []

while ((match = stepPattern.exec(todosSection)) !== null) {
  const stepTitle = match[2].trim()
  const stepContent = match.input.substring(match.index, match.index + match[0].length)

  // 提取验收标准
  const acMatch = stepContent.match(/\*\*Acceptance Criteria\*\*:\s*\n([\s\S]*?)(?=\n\*\*|\n###)/)
  const acceptanceCriteria = acMatch ? acMatch[1].split('- [').map(s => s.trim()) : []

  // 提取文件列表
  const filesMatch = stepContent.match(/\*\*Files to (Create|Modify)\*\*:\s*\n([\s\S]*?)(?=\n\*\*|\n###)/)
  const files = filesMatch ? filesMatch[2].split('\n').map(f => f.trim()) : []

  steps.push({ number: match[1], title: stepTitle, acceptanceCriteria, files })
}
```

### 4. 生成任务

```javascript
const maxId = getMaxTaskId()  // 从 existing tasks 获取
const tasks = []

for (const step of steps) {
  const taskId = `task-${String(maxId + tasks.length + 1).padStart(3, '0')}`

  // 估算优先级（第一步或包含 core/critical = P0）
  let priority = 'P1'
  if (step.number === 1 || /core|critical/i.test(step.title)) priority = 'P0'
  else if (/optional|enhancement/i.test(step.title)) priority = 'P2'

  // 估算时间（基于验收标准和文件数量）
  const hours = Math.max(1, Math.ceil(
    step.acceptanceCriteria.length * 0.5 + step.files.length * 0.3
  ))

  // 确定模块
  let module = 'general'
  if (/test/i.test(step.title)) module = 'testing'
  else if (/doc/i.test(step.title)) module = 'documentation'
  else if (step.files.some(f => /agents|commands/.test(f))) module = 'plugin-core'

  const description = `**来自 OMC 计划**: ${metadata.title}\n\n` +
    `**步骤**: ${step.number}. ${step.title}\n\n` +
    `**验收标准**:\n${step.acceptanceCriteria.map(ac => `- [ ] ${ac}`).join('\n')}\n\n` +
    `**相关文件**:\n${step.files.map(f => `- ${f}`).join('\n')}\n\n` +
    `**计划来源**: .omc/plans/${path.basename(planPath)}`

  tasks.push({ id: taskId, title: step.title, description, priority, module, hours,
              dependencies: tasks.length > 0 ? [tasks[tasks.length - 1].id] : [] })
}
```

### 5. 创建文件

```javascript
// 创建任务卡片
for (const task of tasks) {
  const taskPath = `docs/todo/backlog/${task.id}-${slugify(task.title)}.md`
  const taskContent = `# ${task.id}: ${task.title}
**创建时间**: ${new Date().toISOString().slice(0, 10)}
**优先级**: ${task.priority}
**模块**: ${task.module}
**预计时间**: ${task.hours}h

## 任务描述
${task.description}

## 依赖任务
${task.dependencies.length > 0 ? task.dependencies.join('\n') : '无'}
`

  writeFile(taskPath, taskContent)
}

// 更新 current-sprint.md
updateCurrentSprint(tasks)
```

### 6. 归档 OMC 计划（可选）

```javascript
// 标记为已转换
let archivedContent = content
if (!content.includes('**Converted to Tasks**')) {
  archivedContent = `---
**Converted to Tasks**: ✅
**Conversion Date**: ${new Date().toISOString().slice(0, 10)}
**Generated Tasks**: ${tasks.map(t => t.id).join(', ')}
---

${content}`
}

const archivePath = `docs/done/${new Date().toISOString().slice(0, 7)}/omc-${path.basename(planPath)}`
writeFile(archivePath, archivedContent)
unlink(planPath)  // 删除原计划文件
```

## 输出

```markdown
## 🎯 OMC 计划转换完成

**计划文件**: .omc/plans/omc-integration.md

### 生成任务
✅ **task-001**: Update agent frontmatter (P1, 0.5h)
✅ **task-002**: Create skills directory (P1, 0.75h)
✅ **task-003**: Create AGENTS.md (P2, 0.25h)

**总计**: 6 个任务
**总预计时间**: 3.5 小时

### 下一步
1. 查看任务列表: `cat docs/todo/current-sprint.md`
2. 开始第一个任务: `cat docs/todo/backlog/task-001-*.md`
```

## 错误处理

| 场景 | 处理 |
|--------|--------|
| 文件不存在 | 提示检查路径: `ls .omc/plans/` |
| 无 Detailed TODOs | 提示确保计划包含 ## Detailed TODOs 章节 |
| 已转换过 | 询问是否重新转换 |

## 配置

```yaml
priority_rules:
  first_step: P0
  critical_keywords: [core, critical, important]
  default: P1

time_estimation:
  base_hours: 1
  per_criterion: 0.5
  per_file: 0.3

module_classification:
  agents: plugin-core
  commands: plugin-core
  test: testing
  doc: documentation
  default: general
```
