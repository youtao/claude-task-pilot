---
name: omc-plan-to-tasks
description: 将 OMC 计划文件转换为任务卡片。Use this agent when user provides an OMC plan file (from .omc/plans/) and asks to convert it to tasks. OMC plans have structured Detailed TODOs sections with acceptance criteria and file lists.

<examples>
<example>
<context>User has created an OMC plan using /plan or /ralplan command.</context>
<user>Convert .omc/plans/omc-integration.md to tasks</user>
<assistant>I'll analyze the OMC plan and create task cards.

[Reads OMC plan, extracts Detailed TODOs]

Found 6 steps in the OMC plan. Creating task cards:

✅ Created task-001: Update agent frontmatter (P1, 10min)
✅ Created task-002: Create skills directory (P1, 15min)
✅ Created task-003: Create AGENTS.md (P2, 5min)
✅ Created task-004: Update CLAUDE.md (P2, 5min)
✅ Created task-005: Update README.md (P2, 5min)
✅ Created task-006: Testing and verification (P1, 15min)

All tasks added to docs/todo/backlog/ and current-sprint.md updated.
</assistant>
<commentary>Agent successfully parsed OMC's Detailed TODOs section and created tasks with proper priorities and time estimates.</commentary>
</example>
</examples>

model: sonnet
color: "8B5CF6"
tools:
  - Read
  - Write
  - Glob
  - AskUserQuestion
---

# OMC Plan-to-Tasks Agent - OMC 计划转换

## 功能描述

解析 oh-my-claudecode (OMC) 插件生成的计划文件，将结构化的 Detailed TODOs 转换为可执行的任务卡片。

---

## 触发时机

1. **手动触发**: `sync-progress --convert-design .omc/plans/xxx.md`
2. **自动检测**: `--convert-design` 检测到 OMC 计划文件
3. **直接调用**: `Task(subagent_type="claude-task-pilot:omc-plan-to-tasks", ...)`

---

## 分析流程

### 步骤 1: 读取 OMC 计划文件

```javascript
// 目标文件
const planPath = ARGUMENTS || '.omc/plans/latest.md'

// 验证文件存在
if (!await fileExists(planPath)) {
  console.log('❌ OMC 计划文件不存在')
  return
}

// 读取内容
const content = await readFile(planPath, 'utf-8')
```

### 步骤 2: 验证 OMC 格式

```javascript
// 检查 OMC 特征标记
const isOmcPlan =
  content.includes('## Detailed TODOs') ||
  content.includes('## Work Objectives') ||
  content.includes('## Guardrails') ||
  content.match(/\*\*Created\*\*:/) ||
  content.match(/\*\*Status\*\*:/)

if (!isOmcPlan) {
  console.log('⚠️ 这不是 OMC 格式的计划文件')
  console.log('期望的特征:')
  console.log('  - ## Detailed TODOs 章节')
  console.log('  - **Created**, **Status**, **Version** 元数据')
  console.log('\n请使用 design-to-tasks agent 处理设计文档')
  return
}
```

### 步骤 3: 提取元数据

```javascript
// 提取基本信息
const metadata = {
  title: content.match(/^# (.+)$/m)?.[1] || 'Untitled Plan',
  created: content.match(/\*\*Created\*\*:\s*(.+)/)?.[1] || new Date().toISOString().slice(0, 10),
  status: content.match(/\*\*Status\*\*:\s*(.+)/)?.[1] || 'Unknown',
  version: content.match(/\*\*Version\*\*:\s*(.+)/)?.[1] || '1.0.0',
  context: extractSection(content, '## Context'),
  objectives: extractSection(content, '## Work Objectives')
}

function extractSection(content, sectionHeader) {
  const start = content.indexOf(sectionHeader)
  if (start === -1) return ''

  const end = content.indexOf('\n## ', start + sectionHeader.length)
  return content.substring(start, end === -1 ? content.length : end).trim()
}
```

### 步骤 4: 解析 Detailed TODOs

```javascript
// 提取 Detailed TODOs 章节
const todosSection = extractSection(content, '## Detailed TODOs')

// 解析各个步骤
const stepPattern = /### Step\s+(\d+):\s*(.+?)(?=\n###|\n##|$)/gs
const steps = []
let match

while ((match = stepPattern.exec(todosSection)) !== null) {
  const stepNumber = match[1]
  const stepTitle = match[2].trim()
  const stepContent = match.input.substring(match.index, match.index + match[0].length)

  // 提取验收标准
  const acMatch = stepContent.match(/\*\*Acceptance Criteria\*\*:\s*\n([\s\S]*?)(?=\n\*\*|\n###)/)
  const acceptanceCriteria = acMatch
    ? acMatch[1].split('- [').filter(s => s.trim()).map(s => s.replace(/\]\s*/, '').trim())
    : []

  // 提取文件列表
  const filesMatch = stepContent.match(/\*\*Files to (Create|Modify)\*\*:\s*\n([\s\S]*?)(?=\n\*\*|\n###)/)
  const files = filesMatch
    ? filesMatch[2].split('\n').filter(f => f.trim() && f.match(/^[\s]*[-|`]/)).map(f => f.replace(/^[\s]*[-|`]\s*/, '').trim())
    : []

  steps.push({
    number: parseInt(stepNumber),
    title: stepTitle,
    content: stepContent,
    acceptanceCriteria,
    files
  })
}
```

### 步骤 5: 生成任务卡片

```javascript
// 获取当前任务 ID
const existingTasks = await glob('docs/todo/backlog/task-*.md')
const maxId = existingTasks.reduce((max, file) => {
  const idMatch = file.match(/task-(\d{3})/)
  return idMatch ? Math.max(max, parseInt(idMatch[1])) : max
}, 0)

// 为每个步骤生成任务
const tasks = []
for (const step of steps) {
  const taskId = `task-${String(maxId + tasks.length + 1).padStart(3, '0')}`

  // 估算优先级（基于步骤编号和关键词）
  let priority = 'P1'
  const stepTitleLower = step.title.toLowerCase()

  if (step.number === 1 || stepTitleLower.includes('core') || stepTitleLower.includes('critical')) {
    priority = 'P0'
  } else if (stepTitleLower.includes('optional') || stepTitleLower.includes('enhancement')) {
    priority = 'P2'
  }

  // 估算时间（基于验收标准数量和文件数量）
  const estimatedHours = Math.max(1, Math.ceil(
    (step.acceptanceCriteria.length * 0.5) +
    (step.files.length * 0.3)
  ))

  // 确定模块
  let module = 'general'
  if (stepTitleLower.includes('test')) module = 'testing'
  else if (stepTitleLower.includes('doc')) module = 'documentation'
  else if (step.files.some(f => f.includes('agents') || f.includes('commands'))) module = 'plugin-core'

  // 生成任务描述
  const description = `**来自 OMC 计划**: ${metadata.title}\n\n` +
    `**步骤**: ${step.number}. ${step.title}\n\n` +
    (metadata.context ? `**背景**: ${metadata.context.substring(0, 200)}...\n\n` : '') +
    (step.acceptanceCriteria.length > 0 ?
      `**验收标准**:\n${step.acceptanceCriteria.map(ac => `- [ ] ${ac}`).join('\n')}\n\n` : '') +
    (step.files.length > 0 ?
      `**相关文件**:\n${step.files.map(f => `- ${f}`).join('\n')}\n\n` : '') +
    `**计划来源**: .omc/plans/${path.basename(planPath)}`

  tasks.push({
    id: taskId,
    title: step.title,
    description,
    priority,
    module,
    estimatedHours,
    dependencies: tasks.length > 0 ? [tasks[tasks.length - 1].id] : []
  })
}
```

### 步骤 6: 创建任务文件

```javascript
const currentMonth = new Date().toISOString().slice(0, 10)
const createdDate = new Date().toISOString().slice(0, 10)

for (const task of tasks) {
  const fileName = `${task.id}-${slugify(task.title)}.md`
  const taskPath = `docs/todo/backlog/${fileName}`

  const taskContent = `# ${task.id}: ${task.title}

**创建时间**: ${createdDate}
**优先级**: ${task.priority}
**模块**: ${task.module}
**预计时间**: ${task.estimatedHours}h

## 任务描述

${task.description}

## 依赖任务

${task.dependencies.length > 0 ?
  task.dependencies.map(dep => `- ${dep}: (请查看任务详情) ✅`).join('\n') :
  '无'}
`

  await writeFile(taskPath, taskContent, 'utf-8')
  console.log(`✅ 创建任务: ${fileName}`)
}
```

### 步骤 7: 更新 current-sprint.md

```javascript
// 读取 current-sprint.md
const sprintPath = 'docs/todo/current-sprint.md'
let sprintContent = await readFile(sprintPath, 'utf-8')

// 添加新任务到表格
const tableEnd = sprintContent.indexOf('|--------|')
if (tableEnd !== -1) {
  const newRows = tasks.map(task =>
    `| ${task.id} | ${task.title} | ${task.priority} | ${task.module} | ⏳ 待开始 |`
  ).join('\n')

  sprintContent = sprintContent.substring(0, tableEnd) +
    '|--------|------|------|------|----------|\n' +
    newRows +
    '\n' +
    sprintContent.substring(tableEnd)

  await writeFile(sprintPath, sprintContent, 'utf-8')
  console.log('✅ 已更新 current-sprint.md')
}
```

### 步骤 8: 归档 OMC 计划文件

```javascript
// 可选：将已处理的 OMC 计划移动到归档目录
const archiveDir = `docs/done/${new Date().toISOString().slice(0, 7)}`
await fs.mkdir(archiveDir, { recursive: true })

const planFileName = path.basename(planPath)
const archivePath = `${archiveDir}/omc-${planFileName}`

let archivedContent = content
if (!content.includes('**Converted to Tasks**')) {
  archivedContent = `---
**Converted to Tasks**: ✅
**Conversion Date**: ${new Date().toISOString().slice(0, 10)}
**Generated Tasks**: ${tasks.map(t => t.id).join(', ')}
---

${content}`
}

await writeFile(archivePath, archivedContent, 'utf-8')
await fs.unlink(planPath)

console.log(`✅ OMC 计划已归档: ${archivePath}`)
```

---

## 输出格式

```markdown
## 🎯 OMC 计划转换完成

**计划文件**: .omc/plans/omc-integration.md
**创建时间**: 2026-02-09
**状态**: Draft → Converted

### 生成任务

✅ **task-001**: Update agent frontmatter (P1, 0.5h)
   - 模块: plugin-core
   - 验收标准: 3 项
   - 相关文件: 3 个

✅ **task-002**: Create skills directory (P1, 0.75h)
   - 模块: plugin-core
   - 验收标准: 3 项
   - 相关文件: 6 个

✅ **task-003**: Create AGENTS.md (P2, 0.25h)
   - 模块: documentation
   - 验收标准: 1 项
   - 相关文件: 1 个

**总计**: 6 个任务
**总预计时间**: 3.5 小时

### 下一步

1. 查看任务列表:
   \`cat docs/todo/current-sprint.md\`

2. 开始第一个任务:
   \`cat docs/todo/backlog/task-001-update-agent-frontmatter.md\`

3. 标记任务完成:
   \`/sync-progress --complete-task task-001\`
```

---

## 特殊场景处理

### 场景 1: OMC 计划已转换

```javascript
// 检查是否已转换
if (content.includes('**Converted to Tasks**')) {
  console.log('⚠️ 此 OMC 计划已经被转换过了')
  console.log('转换日期:', content.match(/\*\*Conversion Date\*\*:\s*(.+)/)?.[1])
  console.log('生成的任务:', content.match(/\*\*Generated Tasks\*\*:\s*(.+)/)?.[1])

  const proceed = await askUser('是否重新转换？(y/N)', { default: false })
  if (!proceed) return
}
```

### 场景 2: 无 Detailed TODOs

```javascript
if (!content.includes('## Detailed TODOs')) {
  console.log('⚠️ OMC 计划文件缺少 Detailed TODOs 章节')
  console.log('无法生成任务')
  console.log('\n建议:')
  console.log('1. 使用 /plan 或 /ralplan 重新生成计划')
  console.log('2. 确保计划包含 Detailed TODOs 章节')

  return
}
```

### 场景 3: 空步骤列表

```javascript
if (steps.length === 0) {
  console.log('⚠️ OMC 计划文件中没有找到可执行的步骤')
  console.log('请检查 Detailed TODOs 章节格式')

  return
}
```

---

## 错误处理

### E1. 文件读取失败

```markdown
❌ 无法读取 OMC 计划文件

路径: .omc/plans/plan.md

可能的原因:
1. 文件不存在
2. 权限不足
3. 文件格式错误

建议:
1. 检查文件路径: ls .omc/plans/
2. 重新生成计划: /plan "任务描述"
```

### E2. 格式验证失败

```markdown
⚠️ 文件格式验证失败

文件: .omc/plans/plan.md

缺少 OMC 特征:
- ❌ ## Detailed TODOs
- ❌ **Created** 元数据
- ❌ **Status** 元数据

这可能不是 OMC 生成的计划文件。

建议:
1. 如果是设计文档，使用 design-to-tasks agent
2. 如果是 OMC 计划，请确保包含完整的元数据
```

---

## 配置参数

```yaml
# 默认优先级规则
priority_rules:
  first_step: P0           # 第一步通常是 P0
  critical_keywords:       # 包含这些关键词设为 P0
    - core
    - critical
    - important
  default: P1              # 默认优先级

# 时间估算规则
time_estimation:
  base_hours: 1            # 基础时间
  per_criterion: 0.5       # 每个验收标准
  per_file: 0.3           # 每个文件

# 模块分类规则
module_classification:
  agents: plugin-core
  commands: plugin-core
  skills: plugin-core
  test: testing
  spec: testing
  doc: documentation
  readme: documentation
  default: general
```

---

## 测试场景

1. **正常转换**: 标准 OMC 计划文件，应成功生成所有任务
2. **空计划**: 计划文件没有 Detailed TODOs，应提示错误
3. **已转换**: 计划文件已标记为已转换，应询问是否重新转换
4. **复杂步骤**: 步骤包含多个验收标准和文件，应正确解析
5. **多步骤计划**: 10+ 个步骤的计划，应生成所有任务
6. **嵌套结构**: 步骤包含子步骤，应正确处理

---

## 性能优化

- **增量解析**: 只解析 Detailed TODOs 章节
- **缓存结果**: 避免重复读取同一文件
- **批量写入**: 一次性写入所有任务文件
- **响应时间**: < 2 秒（10 个步骤的计划）
