# Claude Task Pilot - AI 驱动的任务管理系统

## 📖 项目概述

**Claude Task Pilot** 是专为 Claude Code 设计的自动化任务管理插件，通过文件系统监控和智能推荐，为 AI 驱动的开发流程提供无缝的任务追踪和状态管理。

### 核心特性

- **AI 原生设计**: 文档结构化、高信息密度，优化 Claude Code 的理解能力
- **手动触发**: 通过命令精确控制何时执行操作，避免不稳定的自动触发
- **智能推荐**: 基于优先级、依赖关系和里程碑自动推荐下一个任务
- **设计文档转换**: 手动将设计文档转换为可执行的任务卡片
- **快速启动**: 5-10 秒快速了解当前项目状态
- **文档驱动**: 基于文件系统的简单、透明设计

---

## 🏗️ 项目结构

```
claude-task-pilot/
├── .claude-plugin/
│   └── plugin.json              # 插件配置清单
│   └── marketplace.json         # Marketplace 配置
├── agents/
│   ├── task-suggester.md        # 智能任务推荐代理
│   ├── design-to-tasks.md       # 设计文档转换代理
│   └── daily-reporter.md        # 日报生成代理
├── commands/
│   ├── setup-task-pilot.md      # 初始化命令
│   └── sync-progress.md         # 同步进度 + 完成任务 + 转换设计
├── templates/
│   ├── session.md               # Session 状态模板
│   ├── roadmap.md               # 长期路线图模板
│   ├── current-sprint.md        # 当前冲刺模板
│   ├── archive-index.md         # 归档索引模板
│   ├── task-complete.md         # 任务完成报告模板
│   └── task-backlog.md          # 任务卡片模板
├── CLAUDE.md                     # 本文档（AI 指导文档）
└── README.md                     # 用户文档
```

---

## 🎯 核心概念

### 文档结构

插件在用户项目中创建以下结构：

```
docs/
├── session.md              # 当前 Session 状态（<80 行）
├── plans/                  # 设计文档目录
│   └── YYYY-MM-DD-*-design.md
├── todo/
│   ├── roadmap.md          # 长期路线图（3-6 个月）
│   ├── current-sprint.md   # 当前冲刺（1-2 周）
│   └── backlog/            # 任务卡片
│       └── task-XXX-*.md
└── done/                   # 已完成任务和设计归档
    ├── archive-index.md    # 归档索引
    └── YYYY-MM/            # 按月归档
        ├── task-XXX-*.md   # 已完成任务
        └── *-design.md     # 已完成设计
```

### 任务卡片格式

```markdown
# task-003: 任务名称

**创建时间**: 2026-01-30
**优先级**: P0
**模块**: backend
**里程碑**: Phase 2: 核心功能

## 任务描述
简要描述任务内容...

## 验收标准
- [ ] 标准1
- [ ] 标准2

## 依赖任务
- task-002: 依赖任务名称 ✅
```

### 优先级系统

- **P0**: 紧急重要，必须立即处理
- **P1**: 重要但不紧急，本周内完成
- **P2**: 可以延后，有空时处理

---

## 🔧 架构设计

### 命令执行流程

```
/sync-progress
    ↓
检测并修复文档不一致
    ↓
├─ quick: 快速同步（默认）
├─ full: 完全同步（更新 roadmap.md）
└─ verify: 验证模式（不修改文件）

/sync-progress --complete-task
    ↓
归档任务
    ↓
├─ 更新 session.md、current-sprint.md
├─ 归档关联设计文档
└─ 推荐下一个任务

/sync-progress --convert-design
    ↓
转换设计文档
    ↓
├─ 调用 design-to-tasks agent
└─ 生成任务卡片到 backlog/
```

### Agent 调用

#### task-suggester
**触发时机**:
- 使用 `/sync-progress --complete-task` 完成任务后
- 用户主动询问

**推荐逻辑**:
1. 优先级评分（P0: 100, P1: 70, P2: 40）
2. 依赖关系检查（依赖任务必须完成）
3. 里程碑对齐（当前里程碑优先）
4. 模块平衡（避免连续做同一模块）

#### design-to-tasks
**触发时机**:
- 使用 `/sync-progress --convert-design` 时

**转换逻辑**:
1. 分析设计文档内容
2. 按模块拆分任务（backend/frontend/testing/documentation）
3. 自动判断优先级
4. 识别依赖关系

#### daily-reporter
**触发时机**:
- 用户主动调用

**报告内容**:
- 昨日完成任务
- 今日推荐任务
- 本周进度概览

---

## 💡 设计原则

### 1. AI 优先
- 文档使用结构化 Markdown
- 高信息密度，减少 Claude 的 token 消耗
- 明确的状态标记（✅、🔄、⏳）

### 2. 快速启动
- session.md 限制 80 行，快速读取
- 关键信息在前 20 行
- 5-10 秒了解项目状态

### 3. 任务驱动
- 所有开发活动围绕任务卡片
- 任务卡片是真理的唯一来源
- 完成即归档，不可修改

### 4. 完整追溯
- 任务生命周期完整记录
- 按月归档，便于回顾
- archive-index.md 提供全局视图

### 5. 手动触发
- 通过命令精确控制操作时机
- 避免不稳定的自动触发
- 失败时不影响正常开发

---

## 🛠️ 开发指南

### 添加新 Agent

1. 在 `agents/` 目录创建 `.md` 文件
2. 定义触发条件和分析逻辑
3. 在需要时通过命令调用

### 修改模板

模板文件位于 `templates/` 目录，修改后自动生效。

建议：
- 保持简洁，信息密度高
- 使用明确的 emoji 标记
- 包含使用示例

---

## 📄 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

