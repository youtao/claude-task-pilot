---
description: 基于当前项目状态智能推荐下一个任务
color: "8B5CF6"
---

# Task Suggester Agent

## 功能

分析项目状态，推荐最合适的下一个任务。

**推荐依据**:
- 优先级（P0 > P1 > P2）
- 依赖关系（依赖任务已完成）
- 里程碑对齐（当前里程碑优先）
- 模块平衡（避免连续做同模块）

## 触发

- **任务完成后**: 自动推荐
- **用户询问**: "下一个任务" / "推荐任务"

## 流程

### 1. 收集状态

```javascript
// 读取文件
const currentSprint = readFile('docs/todo/current-sprint.md')
const roadmap = readFile('docs/todo/roadmap.md')
const session = readFile('docs/session.md')

// 提取候选任务（未完成且依赖已满足）
const candidates = parseTasks(currentSprint).filter(task => {
  if (task.status === '✅') return false
  return task.dependencies.every(dep => completedTasks.includes(dep))
})
```

### 2. 计算分数

```javascript
candidates.forEach(task => {
  let score = 0

  // 优先级（50%）
  score += { P0: 100, P1: 70, P2: 40 }[task.priority] * 0.5

  // 依赖完成度（30%）
  const depRatio = task.dependencies.filter(d => completedTasks.includes(d)).length /
                   task.dependencies.length
  score += depRatio * 100 * 0.3

  // 里程碑对齐（20%）
  if (task.milestone === currentMilestone) score += 20

  // 模块平衡（惩罚）
  const recentCount = recentTasks.filter(t => t.module === task.module).length
  score -= recentCount * 5

  task.score = score
})
```

### 3. 返回 Top 3

```javascript
const recommended = candidates
  .sort((a, b) => b.score - a.score)
  .slice(0, 3)
  .map(task => ({
    id: task.id,
    name: task.name,
    reason: getReason(task),  // "高优先级，无依赖" / "依赖任务刚完成"
    priority: task.priority,
    estimatedTime: task.estimatedTime
  }))
```

## 输出格式

```markdown
## 🎯 建议的下一个任务

### 推荐 1: task-003 - 补充番茄品种数据
- **理由**: 高优先级（P0），所有依赖任务已完成
- **优先级**: P0
- **预计时间**: 2-3 小时

### 推荐 2: task-004 - 补充生长阶段数据
- **理由**: 同样是 P0，无依赖，可并行准备
- **优先级**: P0

### 推荐 3: task-005 - 设计知识库架构
- **理由**: 里程碑关键路径任务
- **优先级**: P1
```

## 特殊场景

| 场景 | 输出 |
|--------|--------|
| 无可用任务 | "所有任务已完成，查看 roadmap 开始下一里程碑" |
| 所有任务被阻塞 | "等待依赖任务完成: task-001, task-002" |

## 配置

```yaml
max_recommendations: 3
priority_weights:
  P0: 100
  P1: 70
  P2: 40
score_weights:
  priority: 0.5
  dependencies: 0.3
  milestone: 0.2
balance_penalty: 5
```
