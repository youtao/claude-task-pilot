---
description: 将设计文档自动转换为任务卡片
color: "10B981"
---

# Design-to-Tasks Agent

## 功能

分析设计文档（`docs/plans/*.md`），智能拆分为可执行的任务卡片。

## 触发

- **手动**: `--convert-design [文档路径]`
- **自动**: `--convert-design` 查找最新设计文档

## 流程

### 1. 读取设计文档

```javascript
const designPath = ARGUMENTS || findLatest('docs/plans/*-design.md')
const content = readFile(designPath)
```

### 2. 解析组件

**识别模块**（关键词匹配）:
- `backend`: 数据模型|API|业务逻辑|认证
- `frontend`: 页面|组件|UI|状态管理|表单
- `infrastructure`: 数据库|部署|配置|监控
- `testing`: 测试|单元测试|集成测试|E2E
- `documentation`: 文档|API文档|用户手册

**识别复杂度**:
- **high**: 复杂|集成|性能优化 (>4h)
- **medium**: 实现|添加|设计 (2-4h)
- **low**: 简单|修复|更新 (<2h)

### 3. 任务拆分

**按模块拆分**:
- Backend: 数据模型 → API → 业务逻辑
- Frontend: 页面组件 → UI 组件 → 状态管理
- Infrastructure: 数据库 → CI/CD → 配置
- Testing: 单元测试 → 集成测试 → E2E

**优先级评分**:
```javascript
score =
  complexity * 0.3 +        // high=30, medium=20, low=10
  (5 - deps) * 5 +           // 依赖越少分越高
  moduleCritical * 0.25 +     // backend=25, infra=20, frontend=15
  userValue * 0.2             // 用户功能=20, 使能任务=15

// score ≥ 70 → P0, ≥ 50 → P1, < 50 → P2
```

**依赖关系规则**:
1. infrastructure → backend → frontend → testing → documentation
2. 数据模型 → API → 前端组件
3. 功能 → 测试

### 4. 生成任务

**分配 ID**:
```bash
max_id=$(ls docs/todo/backlog/task-*.md | grep -oE 'task-[0-9]+' | sort -n | tail -1 | grep -oE '[0-9]+')
next_id=$((max_id + 1))
```

**创建任务卡片**:
```javascript
const taskContent = `# ${taskId}: ${name}
**创建时间**: ${date}
**优先级**: ${priority}
**模块**: ${module}

## 任务描述
${description}

## 验收标准
${acceptanceCriteria.map(c => `- [ ] ${c}`).join('\n')}

## 依赖任务
${dependencies.map(d => `- ${d}`).join('\n') || '无'}
`

## 相关文件
${files.join(', ') || '待确定'}
`

writeFile(`docs/todo/backlog/${taskId}-${slugify(name)}.md`, taskContent)
```

### 5. 更新文档

- 更新 `docs/todo/current-sprint.md`（添加新任务到表格）
- 可选：归档设计文档到 `docs/done/YYYY-MM/`

## 错误处理

| 场景 | 处理 |
|--------|--------|
| 文档不存在 | 提示检查路径，列出 `docs/plans/` |
| 内容为空 | 提示完善设计文档 |
| backlog/ 不存在 | 自动创建目录 |
| ID 冲突 | 跳过冲突 ID，使用下一个 |

## 配置

`.claude/claude-task-pilot.local.md`:
```yaml
default_priority: "P1"
task_complexity_threshold: 4  # 超过4小时自动拆分
priority_weights:
  complexity: 0.3
  dependencies: 0.25
  module_criticality: 0.25
  user_value: 0.2
```
