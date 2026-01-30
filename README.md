# Claude Task Pilot

**AI-driven task management plugin for Claude Code**

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-purple.svg)](https://github.com/anthropics/claude-code)

## 🎯 Why Claude Task Pilot?

Claude Task Pilot is an AI-native task management system designed specifically for Claude Code workflows. Unlike traditional task management tools, it:

- **Thinks like AI**: Document structure optimized for LLM understanding
- **Works automatically**: Zero manual tracking through smart hooks
- **Recommends intelligently**: Priority-based, dependency-aware task suggestions
- **Starts fast**: Project state in 5-10 seconds

## 🎯 核心功能

### 1. 自动初始化
- 检测项目是否已建立任务管理系统
- 自动创建符合规范的文档结构
- 生成模板文件

### 2. 任务状态追踪
- **任务开始**: 检测新任务卡片创建，自动更新状态
- **任务完成**: 检测任务归档，自动更新索引
- **规范守护**: 确保 session.md 不超过 80 行

### 3. 智能功能
- **任务推荐**: 基于优先级和依赖关系推荐下一个任务
- **日报生成**: 自动生成每日进度报告

## 📁 文档结构

```
docs/
├── session.md              # 当前 Session 状态（<80 行）
├── todo/
│   ├── roadmap.md          # 长期路线图（3-6 个月）
│   ├── current-sprint.md   # 当前冲刺（1-2 周）
│   └── backlog/            # 任务卡片
│       └── task-XXX-*.md
└── done/                   # 已完成任务归档
    ├── archive-index.md    # 归档索引
    └── YYYY-MM/            # 按月归档
        └── task-XXX-*.md
```

## 🚀 快速开始

### 安装

```bash
# Clone to plugin directory
cd ~/.claude/plugins
git clone https://github.com/your-username/claude-task-pilot.git

# Or manually copy
cp -r claude-task-pilot ~/.claude/plugins/
```

### 1 分钟演示

```bash
# Create test project
mkdir ~/test-project && cd ~/test-project

# Start Claude Code
claude-code

# Plugin will prompt for initialization, select "Yes"

# Create first task
cat > docs/todo/backlog/task-001-hello-world.md << 'EOF'
# task-001: Hello World

**创建时间**: 2026-01-30
**优先级**: P0
**模块**: test

## 任务描述
测试 Claude Task Pilot 插件

## 验收标准
- [ ] 插件正常工作
EOF

# Check if session.md is automatically updated
cat docs/session.md
```

### 初始化项目

新项目首次启动时，Plugin 会自动提示初始化：

```markdown
## 🚀 检测到新项目

此项目尚未建立任务管理系统。是否立即初始化？

**将会创建**:
- docs/session.md
- docs/todo/roadmap.md
- docs/todo/current-sprint.md
- docs/done/archive-index.md
- docs/todo/backlog/
- docs/done/YYYY-MM/
```

### 创建任务

在 `docs/todo/backlog/` 创建新任务卡片：

```markdown
# task-003: 补充番茄品种数据

**创建时间**: 2026-01-30
**优先级**: P0
**模块**: backend

## 任务描述
补充 50 个常见番茄品种数据...

## 验收标准
- [ ] 至少 50 个品种
- [ ] 每个品种包含完整信息
```

**自动触发**:
- ✅ 更新 `session.md` "当前任务"
- ✅ 更新 `current-sprint.md` 状态为 🔄

### 完成任务

移动任务卡片到 `docs/done/YYYY-MM/`：

```bash
mv docs/todo/backlog/task-003.md docs/done/2026-01/
```

**自动触发**:
- ✅ 创建完成报告
- ✅ 更新 `session.md` "上一个任务"
- ✅ 更新 `current-sprint.md` 状态为 ✅
- ✅ 更新 `archive-index.md`
- ✅ 智能推荐下一个任务

## 🔧 配置

### 项目级配置

在项目根目录创建 `.claude/claude-task-pilot.local.md`：

```yaml
---
docs_root: "docs"
session_max_lines: 80
auto_recommend: true
daily_report: true
---
```

### 全局配置

编辑 `~/.claude/plugins/claude-task-pilot/.claude/claude-task-pilot.local.md`

## 📖 使用指南

### 日常开发流程

1. **启动 Session**: 自动显示当前任务摘要
2. **开始任务**: 创建任务卡片到 `backlog/`
3. **完成任务**: 移动任务卡片到 `done/YYYY-MM/`
4. **查看日报**: 每日首次启动自动生成

### 手动命令

```markdown
# 查看当前状态
→ 显示: session.md 摘要

# 推荐下一个任务
→ 调用: task-suggester agent

# 生成日报
→ 调用: daily-reporter agent
```

## 🏗️ 架构

### Hooks

- **SessionStart**: 初始化检测 + 摘要显示
- **PreToolWrite**: 任务开始检测
- **PostToolWrite**: 任务完成 + 规范守护

### Agents

- **task-suggester**: 智能任务推荐
- **daily-reporter**: 日报生成

### Templates

- `session.md`: Session 状态模板
- `roadmap.md`: 路线图模板
- `current-sprint.md`: 冲刺模板
- `archive-index.md`: 归档索引模板
- `task-complete.md`: 完成报告模板

## 🎨 设计原则

1. **AI 优先**: 文档结构化、高信息密度
2. **快速启动**: 新 Session 5-10 秒了解状态
3. **任务驱动**: 所有开发活动围绕任务卡片
4. **完整追溯**: 任务生命周期记录完整
5. **非侵入式**: Hooks 观察，不干扰用户操作

## 🔧 故障排查

### session.md 损坏

```bash
# Delete corrupted file
rm docs/session.md

# Restart Claude Code, auto-rebuild
claude-code
```

### 归档索引损坏

```bash
# Rebuild from template
cp ~/.claude/plugins/claude-task-pilot/templates/archive-index.md \
   docs/done/archive-index.md
```

### Hook 未触发

```bash
# Check plugin directory
ls ~/.claude/plugins/claude-task-pilot

# Check plugin.json syntax
cat ~/.claude/plugins/claude-task-pilot/.claude-plugin/plugin.json | jq
```

### Agent 调用失败

- 检查 Agent 文件是否存在
- 检查 `plugin.json` 中的 agent 声明
- 查看 Claude Code 日志

## 🧪 测试

### 手动测试场景

创建测试项目验证插件功能：

```bash
# 1. 创建测试项目
mkdir ~/test-pilot && cd ~/test-pilot

# 2. 启动 Claude Code
claude-code

# 3. 验证自动初始化提示
# 4. 创建任务卡片
# 5. 移动任务到归档
# 6. 检查自动更新是否生效
```

**测试清单**：
- [ ] 新项目自动初始化
- [ ] 任务创建后 session.md 自动更新
- [ ] 任务完成后自动生成报告
- [ ] 归档索引自动更新
- [ ] 智能推荐功能正常
- [ ] session.md 行数限制生效

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

### 开发指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 开启 Pull Request

## 📄 License

MIT License - 详见 [LICENSE](LICENSE) 文件

---

**最后更新**: 2026-01-30
**维护者**: youtao
