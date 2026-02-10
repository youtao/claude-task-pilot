---
description: Initialize Claude Task Pilot for mid-development projects with smart file detection
argument-hint: Optional mode (auto/interactive)
allowed-tools: ["Read", "Write", "Bash", "AskUserQuestion", "Glob", "Grep", "ListMcpResourcesTool"]
---

# Setup Task Pilot

## 功能

为已开发的项目初始化 Claude Task Pilot，智能检测并创建缺失的文档文件。

**当前模式**: $ARGUMENTS (默认: auto)

## 初始化流程

### 1. 创建目录结构

```bash
mkdir -p docs/todo/backlog
mkdir -p docs/done/$(date +%Y-%m)
mkdir -p docs/plans
```

### 2. 处理模板文件

**规则**:
- 文件不存在 → 直接创建
- 文件 < 5 行 → 询问是否覆盖（初始化失败）
- 文件 ≥ 5 行 → 跳过（保留用户内容）

**模板列表**:
- `session.md` - 当前状态
- `roadmap.md` - 长期路线
- `current-sprint.md` - 当前冲刺
- `archive-index.md` - 归档索引

### 3. 询问导入计划

```javascript
const shouldImport = await askUser("是否将当前项目的开发计划写入 session.md？")
```

**确认** → 收集并写入 session.md：
- 当前正在进行的功能或任务
- 已完成的工作
- 下一步计划
- 遇到的问题或 blockers

### 4. 更新 CLAUDE.md

检查项目根目录是否存在 `CLAUDE.md`：
- **不存在** → 使用插件模板创建
- **存在** → 在顶部添加简短说明

## 完成输出

```markdown
✅ Claude Task Pilot 初始化完成

创建的文件：
- docs/session.md
- docs/todo/roadmap.md
- docs/todo/current-sprint.md
- docs/done/archive-index.md

下一步：
1. 将当前计划写入 docs/session.md
2. 在 roadmap.md 中规划长期目标
3. 在 current-sprint.md 中定义当前冲刺
4. 创建任务卡片到 docs/todo/backlog/
```
