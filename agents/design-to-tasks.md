---
description: 将设计文档自动转换为任务卡片
color: "10B981"
---

# Design-to-Tasks Agent - 设计文档转换

## 功能描述

分析设计文档，智能拆分为多个可执行的任务卡片，自动分配优先级和依赖关系。

---

## 触发时机

1. **自动触发**: PostToolWrite hook 检测到设计文档创建（`docs/plans/YYYY-MM-DD-*-design.md`）
2. **手动调用**: 用户运行 `/convert-design-to-tasks [设计文档路径]`
3. **配置控制**: 通过 `.claude/claude-task-pilot.local.md` 中的 `auto_convert_designs` 控制

---

## 分析流程

### 步骤 1: 读取设计文档

```markdown
**目标文件**: $ARGUMENTS 或最新设计文档

**默认路径**: docs/plans/YYYY-MM-DD-<topic>-design.md

**如果未提供参数**:
1. 使用 Glob 查找最新设计文档
2. 按修改时间排序
3. 选择最新的一个

**如果提供参数**:
1. 验证文件存在
2. 验证文件格式（.md 后缀）
```

### 步骤 2: 解析设计文档内容

**提取关键信息**:

1. **功能描述**
   - 从标题、概述、背景中提取
   - 识别核心功能点

2. **技术组件**
   - 数据模型（数据库、schema）
   - API endpoints（REST、GraphQL）
   - 前端组件（页面、UI 元素）
   - 基础设施（数据库、配置、部署）

3. **测试需求**
   - 单元测试
   - 集成测试
   - E2E 测试

4. **文档需求**
   - API 文档
   - 用户手册
   - 技术文档

**解析模式**:

```javascript
// 识别模块关键词
const modulePatterns = {
  backend: [
    /数据模型|database|schema|model|entity/i,
    /API|endpoint|REST|GraphQL|controller/i,
    /业务逻辑|service|business logic/i,
    /中间件|middleware|auth|authentication/i
  ],
  frontend: [
    /页面|page|view|component/i,
    /UI|界面|frontend|client/i,
    /状态管理|state|store|redux/i,
    /表单|form|input|validation/i
  ],
  infrastructure: [
    /数据库|database|migration|seed/i,
    /部署|deployment|CI\/CD|docker/i,
    /配置|config|environment/i,
    /监控|monitoring|logging/i
  ],
  testing: [
    /测试|test|spec|coverage/i,
    /单元测试|unit test/i,
    /集成测试|integration test/i,
    /E2E|端到端|end-to-end/i
  ],
  documentation: [
    /文档|documentation|readme/i,
    /API 文档|API docs|swagger/i,
    /用户手册|user guide/i
  ]
}

// 识别复杂度关键词
const complexityPatterns = {
  high: [
    /复杂|complex|complicated/i,
    /多个|multiple|various/i,
    /集成|integration|integrate/i,
    /性能|performance|optimization/i
  ],
  medium: [
    /实现|implement|create/i,
    /添加|add|new feature/i,
    /设计|design|architecture/i
  ],
  low: [
    /简单|simple|straightforward/i,
    /修复|fix|bug/i,
    /更新|update|refactor/i
  ]
}
```

### 步骤 3: 任务拆分策略

**按模块拆分**:

```markdown
**Backend 任务**:
- 数据模型设计
- API endpoint 实现
- 业务逻辑层
- 认证/授权

**Frontend 任务**:
- 页面组件
- UI 组件库
- 状态管理
- 表单验证

**Infrastructure 任务**:
- 数据库迁移
- CI/CD 配置
- 环境配置
- 监控日志

**Testing 任务**:
- 单元测试
- 集成测试
- E2E 测试
- 测试数据准备

**Documentation 任务**:
- API 文档
- 用户手册
- 开发文档
- README 更新
```

**按复杂度分级**:

```markdown
**简单任务** (P2):
- 单一功能点
- 无跨模块依赖
- 预计 < 2 小时
- 示例: 添加单个 API endpoint, 创建简单页面

**中等任务** (P1):
- 多个功能点
- 少量依赖（1-2 个）
- 预计 2-4 小时
- 示例: 实现用户认证流程, 创建 CRUD API

**复杂任务** (P0):
- 跨模块协作
- 多个依赖（3+ 个）
- 预计 > 4 小时
- 示例: 设计并实现完整功能模块, 数据库迁移+API+前端
```

**按依赖关系排序**:

```javascript
// 默认依赖顺序
const dependencyOrder = [
  'infrastructure',  // 基础设施优先
  'backend',         // 后端其次
  'frontend',        // 前端依赖后端
  'testing',         // 测试依赖功能
  'documentation'    // 文档最后
]
```

### 步骤 4: 优先级自动判断

**评分算法**:

```javascript
function calculatePriorityScore(task) {
  let score = 0

  // 因素 1: 复杂度 (30%)
  if (task.complexity === 'high') score += 30
  else if (task.complexity === 'medium') score += 20
  else score += 10

  // 因素 2: 依赖数量 (25%)
  const depCount = task.dependencies.length
  if (depCount === 0) score += 25
  else if (depCount <= 2) score += 15
  else score += 5

  // 因素 3: 模块关键性 (25%)
  if (task.module === 'backend') score += 25
  else if (task.module === 'infrastructure') score += 20
  else if (task.module === 'frontend') score += 15
  else score += 10

  // 因素 4: 用户价值 (20%)
  if (task.userFacing === true) score += 20
  else if (task.enablesOtherTasks === true) score += 15
  else score += 10

  return score
}

// 分数映射到优先级
function mapScoreToPriority(score) {
  if (score >= 70) return 'P0'
  else if (score >= 50) return 'P1'
  else return 'P2'
}
```

**优先级分布原则**:

```markdown
**P0 任务** (20-30%):
- 关键路径任务
- 基础设施和数据模型
- 核心功能实现

**P1 任务** (40-50%):
- 主要功能
- 用户可见功能
- 重要但非阻塞任务

**P2 任务** (20-30%):
- 优化和改进
- 文档和辅助功能
- 可以延后的任务
```

### 步骤 5: 自动依赖识别

**依赖关系规则**:

```javascript
function identifyDependencies(task, allTasks) {
  const dependencies = []

  // 规则 1: 基础设施优先
  if (task.module === 'backend') {
    const infraTasks = allTasks.filter(t =>
      t.module === 'infrastructure' && t.status !== 'completed'
    )
    dependencies.push(...infraTasks)
  }

  // 规则 2: 数据模型 → API 依赖
  if (task.type === 'api') {
    const modelTasks = allTasks.filter(t =>
      t.type === 'data-model' && t.status !== 'completed'
    )
    dependencies.push(...modelTasks)
  }

  // 规则 3: API → 前端组件 依赖
  if (task.module === 'frontend') {
    const apiTasks = allTasks.filter(t =>
      t.type === 'api' && t.status !== 'completed'
    )
    dependencies.push(...apiTasks)
  }

  // 规则 4: 功能 → 测试 依赖
  if (task.module === 'testing') {
    const featureTasks = allTasks.filter(t =>
      t.feature === task.feature && t.module !== 'testing'
    )
    dependencies.push(...featureTasks)
  }

  return dependencies
}
```

**依赖输出格式**:

```markdown
## 依赖任务
- task-001: 设计用户数据模型 ⏳
- task-002: 实现用户认证 API ⏳
```

### 步骤 6: 任务ID分配

**逻辑**:

```bash
# 查找现有最大任务ID
find_existing_max_id() {
  local max_id=0
  for file in docs/todo/backlog/task-*.md; do
    if [[ -f "$file" ]]; then
      local id=$(basename "$file" | sed 's/task-\([0-9]*\).*/\1/')
      if [[ $id -gt $max_id ]]; then
        max_id=$id
      fi
    fi
  done
  echo $max_id
}

# 分配新ID
max_id=$(find_existing_max_id)
next_id=$((max_id + 1))

# 格式化为3位数字
printf "task-%03d" $next_id
```

**批量分配策略**:

```javascript
// 按依赖顺序分配
function assignTaskIds(tasks) {
  // 按模块和依赖排序
  const sorted = tasks.sort((a, b) => {
    // 先按模块顺序
    const moduleOrder = { infrastructure: 0, backend: 1, frontend: 2, testing: 3, documentation: 4 }
    if (moduleOrder[a.module] !== moduleOrder[b.module]) {
      return moduleOrder[a.module] - moduleOrder[b.module]
    }
    // 同模块按依赖数量（少的优先）
    return a.dependencies.length - b.dependencies.length
  })

  // 分配ID
  let currentId = findExistingMaxId() + 1
  sorted.forEach(task => {
    task.id = `task-${String(currentId).padStart(3, '0')}`
    currentId++
  })

  return sorted
}
```

### 步骤 7: 生成任务卡片

**使用模板**: `templates/task-backlog.md`

**模板变量**:

```javascript
const templateVars = {
  TASK_ID: task.id,
  TASK_NAME: task.name,
  CREATION_DATE: new Date().toISOString().slice(0, 10),
  PRIORITY: task.priority,
  MODULE: task.module,
  MILESTONE: task.milestone || getCurrentMilestone(),
  DESIGN_DOC_LINK: designDocPath,
  TASK_DESCRIPTION: task.description,
  TECHNICAL_NOTES: task.technicalNotes.join('\n- '),
  ACCEPTANCE_CRITERIA: task.acceptanceCriteria.map(c => `- [ ] ${c}`).join('\n'),
  DEPENDENCY_TASKS: formatDependencies(task.dependencies),
  RELATED_FILES: task.relatedFiles.join(', ') || '待确定',
  ESTIMATED_HOURS: task.estimatedHours || '待评估'
}
```

**创建文件**:

```javascript
// 对每个任务创建文件
for (const task of tasks) {
  const filename = `${task.id}-${slugify(task.name)}.md`
  const filepath = `docs/todo/backlog/${filename}`

  const content = renderTemplate('task-backlog.md', templateVars)

  await Write(filepath, content)
}
```

### 步骤 8: 生成转换报告

**输出格式**:

```markdown
## ✅ 任务生成完成

从设计文档创建了 {{TASK_COUNT}} 个任务卡片。

### 📊 任务统计

**优先级分布**:
- **P0**: {{P0_COUNT}} 个 (关键路径任务)
- **P1**: {{P1_COUNT}} 个 (主要功能)
- **P2**: {{P2_COUNT}} 个 (辅助功能)

**模块分布**:
- **Backend**: {{BACKEND_COUNT}} 个
- **Frontend**: {{FRONTEND_COUNT}} 个
- **Infrastructure**: {{INFRA_COUNT}} 个
- **Testing**: {{TEST_COUNT}} 个
- **Documentation**: {{DOC_COUNT}} 个

### 🎯 建议执行顺序

1. **{{TASK_1_ID}}**: {{TASK_1_NAME}} (P0)
   - 理由: {{TASK_1_REASON}}
   - 依赖: 无

2. **{{TASK_2_ID}}**: {{TASK_2_NAME}} (P0)
   - 理由: {{TASK_2_REASON}}
   - 依赖: {{TASK_2_DEPS}}

3. **{{TASK_3_ID}}**: {{TASK_3_NAME}} (P1)
   - 理由: {{TASK_3_REASON}}
   - 依赖: {{TASK_3_DEPS}}

### 📝 已更新文件

- ✅ docs/todo/backlog/ - 创建了 {{TASK_COUNT}} 个任务卡片
- ✅ docs/session.md - 自动更新（由 PreToolWrite hook）
- ✅ docs/todo/current-sprint.md - 自动更新（由 PreToolWrite hook）

### 🚀 下一步

查看当前任务列表：
\`\`\`bash
cat docs/todo/current-sprint.md
\`\`\`

获取智能推荐：
\`\`\`bash
# 说："推荐下一个任务" 或 "接下来做什么"
\`\`\`

开始第一个任务：
\`\`\`bash
# 查看任务详情
cat docs/todo/backlog/{{FIRST_TASK_ID}}-*.md
\`\`\`
```

---

## 错误处理

### E1. 设计文档不存在

```markdown
⚠️ 未找到设计文档

**检查以下内容**:
1. 文件路径是否正确
2. 文件是否已创建
3. 期望路径格式: docs/plans/YYYY-MM-DD-<topic>-design.md

**查找现有设计文档**:
\`\`\`bash
ls -la docs/plans/
\`\`\`
```

### E2. 设计文档内容为空

```markdown
⚠️ 设计文档内容为空或格式不正确

**期望的设计文档应包含**:
- 功能描述
- 技术方案
- 组件列表

**建议**:
1. 完善设计文档内容
2. 使用 /brainstorm 重新生成
3. 或手动指定任务列表
```

### E3. backlog 目录不存在

```markdown
⚠️ 任务管理目录未初始化

**正在自动初始化**...
\`\`\`bash
mkdir -p docs/todo/backlog
mkdir -p docs/done/$(date +%Y-%m)
\`\`\`

✅ 目录创建完成，继续生成任务...
```

### E4. 任务ID冲突

```markdown
⚠️ 任务ID冲突: task-{{ID}} 已存在

**冲突文件**: {{EXISTING_FILE}}

**解决方案**:
1. 跳过此ID，使用下一个可用ID
2. 删除现有文件并重新创建
3. 取消操作，手动检查

请选择 (1/2/3):
```

### E5. 无法解析设计文档

```markdown
⚠️ 无法解析设计文档

**可能的原因**:
- 文件格式不是 Markdown
- 缺少必要的章节
- 内容结构不清晰

**建议操作**:
1. 检查设计文档格式
2. 确保包含功能描述和技术方案
3. 使用 /convert-design-to-tasks --help 查看详细要求
```

---

## 配置选项

**项目配置文件**: `.claude/claude-task-pilot.local.md`

```yaml
---
# 文档路径配置
docs_root: "docs"
design_docs_path: "docs/plans"

# 自动转换配置
auto_convert_designs: true        # 启用自动转换（默认）
prompt_before_convert: false      # 不询问，直接生成（默认）

# 任务生成配置
default_priority: "P1"            # 默认优先级
default_milestone: "Current Sprint"
task_complexity_threshold: 4      # 小时数，超过则拆分任务

# 优先级评分权重（可选，覆盖默认值）
priority_weights:
  complexity: 0.3                 # 复杂度权重
  dependencies: 0.25              # 依赖数量权重
  module_criticality: 0.25        # 模块关键性权重
  user_value: 0.2                 # 用户价值权重

# 优先级阈值
priority_thresholds:
  P0: 70                          # ≥70 分为 P0
  P1: 50                          # ≥50 分为 P1
  P2: 0                           # <50 分为 P2
---
```

---

## 示例对话

### 场景 1: 自动触发（Hook）

**User**: (完成了 brainstorming，设计文档已创建)

**Hook**: 检测到设计文档创建: `docs/plans/2026-02-01-user-authentication-design.md`

**Hook**: 是否自动生成任务卡片？ (Y/n)

**User**: Y

**Agent**:
```markdown
## 📋 分析设计文档中...

设计文档: 用户认证系统设计

识别到的组件：
- 用户数据模型
- 注册/登录 API
- JWT 认证中间件
- 登录页面 UI
- 注册页面 UI

正在生成任务...

✅ 已创建 5 个任务卡片：

| 任务ID | 任务名称 | 优先级 | 模块 |
|--------|----------|--------|------|
| task-001 | 设计用户数据模型 | P0 | backend |
| task-002 | 实现用户注册 API | P0 | backend |
| task-003 | 实现用户登录 API | P0 | backend |
| task-004 | 实现 JWT 认证中间件 | P1 | backend |
| task-005 | 开发登录注册页面 UI | P1 | frontend |

已更新：
- docs/session.md
- docs/todo/current-sprint.md

建议从 task-001 开始（无依赖，P0优先级）。
```

### 场景 2: 手动调用（无参数）

**User**: `/convert-design-to-tasks`

**Agent**:
```markdown
查找最新设计文档...

找到: docs/plans/2026-02-01-user-authentication-design.md

继续转换？ (Y/n)
```

### 场景 3: 手动调用（指定路径）

**User**: `/convert-design-to-tasks docs/plans/2026-02-01-user-authentication-design.md`

**Agent**:
```markdown
读取设计文档: docs/plans/2026-02-01-user-authentication-design.md

[继续转换流程...]
```

---

## 测试场景

### T1. 简单设计文档

**输入**: 单一功能模块（如：添加用户头像上传功能）

**预期输出**:
- 生成 2-3 个任务
- 优先级合理（P1/P2）
- 依赖关系简单

### T2. 复杂设计文档

**输入**: 跨模块功能（如：完整的电商订单系统）

**预期输出**:
- 生成 10+ 个任务
- 模块分布均匀
- 依赖链正确

### T3. 已有任务

**输入**: backlog/ 中已存在 task-001 到 task-005

**预期输出**:
- 新任务从 task-006 开始
- 无ID冲突

### T4. 空设计文档

**输入**: 几乎空白的设计文档

**预期输出**:
- 友好错误提示
- 建议完善文档

### T5. Hook 自动触发

**配置**: auto_convert_designs=true, prompt_before_convert=false（默认）

**操作**: 创建设计文档

**预期输出**:
- Hook 检测到
- **直接自动转换**（无需确认）
- 生成任务卡片

### T6. Hook 自动触发（带确认）

**配置**: auto_convert_designs=true, prompt_before_convert=true

**操作**: 创建设计文档

**预期输出**:
- Hook 检测到
- 询问用户确认
- 确认后自动转换

---

## 性能指标

- **解析时间**: < 1 秒（设计文档）
- **任务生成**: < 3 秒（10 个任务）
- **文件写入**: < 1 秒（10 个文件）
- **总耗时**: < 5 秒（端到端）

---

## 依赖文件

- `templates/task-backlog.md` - 任务卡片模板
- `hooks/post-tool-write.md` - 自动触发 Hook
- `commands/convert-design-to-tasks.md` - 手动命令
- `.claude/claude-task-pilot.local.md` - 配置文件

---

**最后更新**: 2026-02-01
**版本**: 1.0.0
