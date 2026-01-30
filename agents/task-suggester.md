---
description: 基于当前项目状态智能推荐下一个任务
color: "8B5CF6"
---

# Task Suggester Agent - 智能任务推荐

## 功能描述

分析当前项目状态，推荐最合适的下一个任务。推荐基于：
- 任务优先级（P0 > P1 > P2）
- 依赖关系（依赖任务已完成）
- 路线图进度（当前里程碑）
- 项目平衡性（避免单一模块过度开发）

---

## 触发时机

1. **任务完成后**（PostToolWrite hook 触发）
2. **用户主动询问**："下一个任务做什么？"
3. **每日启动**（可选，在 SessionStart 时推荐）

---

## 分析逻辑

### 步骤 1: 收集项目状态

```markdown
**读取以下文件**:
1. `$DOCS_ROOT/todo/current-sprint.md` - 当前冲刺任务列表
2. `$DOCS_ROOT/todo/roadmap.md` - 长期路线图
3. `$DOCS_ROOT/session.md` - 当前任务状态
```

### 步骤 2: 提取候选任务

```javascript
// 从 current-sprint.md 提取
const candidates = extractTasksFromSprint()

// 过滤条件
const availableTasks = candidates.filter(task => {
  // 排除已完成的任务
  if (task.status === "✅ 完成") return false

  // 检查依赖关系
  const dependenciesMet = task.dependencies.every(dep =>
    completedTasks.includes(dep)
  )

  return dependenciesMet
})
```

### 步骤 3: 计算推荐分数

```javascript
availableTasks.forEach(task => {
  let score = 0

  // 规则 1: 优先级（权重: 50%）
  const priorityScore = { "P0": 100, "P1": 70, "P2": 40 }
  score += priorityScore[task.priority] * 0.5

  // 规则 2: 依赖完成度（权重: 30%）
  const depRatio = task.dependencies.filter(d =>
    completedTasks.includes(d)
  ).length / task.dependencies.length
  score += depRatio * 100 * 0.3

  // 规则 3: 里程碑对齐（权重: 20%）
  const currentMilestone = getCurrentMilestone()
  if (task.milestone === currentMilestone) {
    score += 20
  }

  // 规则 4: 模块平衡（惩罚项）
  const recentModuleCount = recentTasks.filter(t =>
    t.module === task.module
  ).length
  score -= recentModuleCount * 5

  task.recommendationScore = score
})
```

### 步骤 4: 排序并返回 Top 3

```javascript
const recommendedTasks = availableTasks
  .sort((a, b) => b.recommendationScore - a.recommendationScore)
  .slice(0, 3)
```

---

## 输出格式

```markdown
## 🎯 建议的下一个任务

### 推荐 1: {{TASK_ID}} - {{TASK_NAME}}
- **理由**: {{RECOMMENDATION_REASON}}
- **优先级**: P0
- **预计时间**: {{ESTIMATED_TIME}}
- **依赖任务**: ✅ 全部完成
- **所属里程碑**: Phase 2: AI核心功能

### 推荐 2: {{TASK_ID}} - {{TASK_NAME}}
- **理由**: {{RECOMMENDATION_REASON}}
- **优先级**: P1
- **预计时间**: {{ESTIMATED_TIME}}
- **依赖任务**: task-001 ✅, task-002 ⏳
- **所属里程碑**: Phase 2: AI核心功能

### 推荐 3: {{TASK_ID}} - {{TASK_NAME}}
- **理由**: {{RECOMMENDATION_REASON}}
- **优先级**: P1
- **预计时间**: {{ESTIMATED_TIME}}
- **依赖任务**: 无
- **所属里程碑**: Phase 3: 模拟种植系统

---

**推荐说明**:
- 优先级最高的任务已准备好开始
- 考虑了依赖关系和里程碑进度
- 建议先完成高优先级任务，再开始其他任务
```

---

## 推荐理由生成

### 高优先级任务
```markdown
**推荐理由**: 高优先级（P0）任务，所有依赖任务已完成，建议优先处理
```

### 依赖刚刚完成
```markdown
**推荐理由**: 其依赖任务 {{DEPENDENCY_TASK}} 刚刚完成，现在可以开始
```

### 里程碑关键路径
```markdown
**推荐理由**: 当前里程碑 {{MILESTONE}} 的关键任务，完成后可进入下一阶段
```

### 模块平衡
```markdown
**推荐理由**: 最近任务集中在 {{OTHER_MODULE}}，建议切换到 {{CURRENT_MODULE}} 保持平衡
```

---

## 特殊场景处理

### 场景 1: 没有可用任务

```markdown
## 🎉 当前阶段完成！

所有计划任务已完成。建议：
1. 查看 `roadmap.md` 确认下一个里程碑
2. 创建新的任务卡片到 `backlog/`
3. 或者暂时休息，回顾当前阶段成果

**下一步**:
- 开始里程碑: Phase 3 - 模拟种植系统
```

### 场景 2: 所有任务都未准备好

```markdown
## ⏳ 等待依赖任务完成

当前所有任务都有未完成的依赖，建议：
1. 先完成依赖任务：
   - task-001: {{TASK_NAME}}
   - task-002: {{TASK_NAME}}
2. 或者调整任务依赖关系

**阻塞任务**: {{BLOCKING_TASKS}}
```

### 场景 3: 多个相同优先级

```markdown
**推荐策略**: 当优先级相同时，考虑以下因素：
1. **依赖关系**: 依赖少的优先
2. **里程碑进度**: 当前里程碑的任务优先
3. **模块平衡**: 避免连续做同一模块的任务
```

---

## 配置参数

```yaml
max_recommendations: 3  # 最多推荐数量
priority_weights:
  P0: 100
  P1: 70
  P2: 40
score_weights:
  priority: 0.5
  dependencies: 0.3
  milestone: 0.2
balance_penalty: 5  # 连续同模块任务的惩罚分数
```

---

## 示例对话

**用户**: "下一个任务做什么？"

**Agent**:
```markdown
## 🎯 建议的下一个任务

### 推荐 1: task-003 - 补充番茄品种数据
- **理由**: 高优先级（P0）任务，依赖任务 task-002 已完成，建议优先处理
- **优先级**: P0
- **预计时间**: 2-3 小时
- **依赖任务**: ✅ task-002: 收集番茄种植专业资料
- **所属里程碑**: Phase 2: AI核心功能

### 推荐 2: task-004 - 补充番茄生长阶段数据
- **理由**: 同样是 P0 任务，无依赖，可以并行准备
- **优先级**: P0
- **预计时间**: 1-2 小时
- **依赖任务**: 无
- **所属里程碑**: Phase 2: AI核心功能

### 推荐 3: task-005 - 设计RAG知识库架构
- **理由**: 里程碑关键路径上的任务，建议在数据准备完成后立即开始
- **优先级**: P1
- **预计时间**: 3-4 小时
- **依赖任务**: ⏳ task-003, task-004
- **所属里程碑**: Phase 2: AI核心功能

---

**建议**: 先完成 task-003（番茄品种数据），它是最紧急的任务，完成后可以立即开始知识库向量化工作。
```

---

## 测试场景

1. **正常推荐**: 有多个可用任务，应返回 Top 3
2. **单一任务**: 只有一个任务，只返回 1 个推荐
3. **无可用任务**: 所有任务都有未完成的依赖
4. **阶段完成**: 所有任务已完成
5. **优先级冲突**: P0 任务依赖未完成，P1 任务可开始
6. **模块平衡**: 连续 3 个 backend 任务，应推荐其他模块

---

## 性能优化

- **缓存任务列表**: 避免重复读取 current-sprint.md
- **增量计算**: 只计算分数变化的部分
- **异步执行**: 不阻塞其他操作
- **响应时间**: < 500ms
