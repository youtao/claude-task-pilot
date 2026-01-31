# Claude Task Pilot

**AI 驱动的 Claude Code 任务管理插件**

[![版本](https://img.shields.io/badge/版本-1.4.0-blue.svg)](https://github.com/youtao/claude-task-pilot)
[![许可证](https://img.shields.io/badge/许可证-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-插件-purple.svg)](https://github.com/anthropics/claude-code)

## 🎯 为什么选择 Claude Task Pilot？

Claude Task Pilot 是专为 Claude Code 工作流设计的 AI 原生任务管理系统。与传统任务管理工具不同，它具有以下特点：

- **AI 思维模式**: 文档结构专为 LLM 理解优化
- **自动工作**: 通过智能 Hooks 零手动追踪
- **智能推荐**: 基于优先级和依赖关系的任务建议
- **快速启动**: 5-10 秒了解项目状态

---

## 🎯 核心功能

### 1. 自动初始化
- 检测项目是否已建立任务管理系统
- 自动创建符合规范的文档结构
- 生成模板文件

### 2. 手动初始化
- **/setup-task-pilot 命令**：中途安装插件的项目的初始化工具
- 智能判断：保护现有数据，补充缺失部分
- 交互模式：对可疑文件询问是否重建
- 参数支持：auto（自动）/ interactive（交互）

### 3. 任务状态追踪
- **任务开始**: 检测新任务卡片创建，自动更新状态
- **任务完成**: 检测任务归档，自动更新索引
- **规范守护**: 确保 session.md 不超过 80 行

### 4. 智能功能
- **任务推荐**: 基于优先级和依赖关系推荐下一个任务
- **日报生成**: 自动生成每日进度报告

---

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

---

## 🚀 快速开始

### 安装

#### 方式 1: Marketplace 安装（推荐）

```bash
# 添加 marketplace
/plugin marketplace add https://github.com/youtao/claude-task-pilot.git

# 安装插件
/plugin install claude-task-pilot@claude-task-pilot
```

#### 方式 2: 直接克隆

```bash
# 克隆到插件目录
cd ~/.claude/plugins
git clone https://github.com/youtao/claude-task-pilot.git
```

### 1 分钟演示

```bash
# 创建测试项目
mkdir ~/test-project && cd ~/test-project

# 启动 Claude Code
claude-code

# 插件会提示初始化，选择"是"

# 创建第一个任务
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

# 检查 session.md 是否自动更新
cat docs/session.md
```

### 初始化项目

#### 新项目初始化

新项目首次启动时，插件会自动提示初始化：

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

#### 中途安装插件

如果你是在已有项目中中途安装此插件，有两种初始化方式：

**方式 1: 使用斜杠命令（推荐）**

```bash
/setup-task-pilot
```

可选参数：
- `auto` - 自动模式（智能判断，少询问）
- `interactive` - 交互模式（询问每个可能覆盖的文件）

**方式 2: 自然语言触发**

```
帮我初始化任务管理结构
```

或

```
设置项目结构用于任务追踪
```

**智能行为**：
- ✅ 自动创建缺失的目录和文件
- 🤖 对于已存在但内容很少的文件（< 5 行），询问是否重新生成
- 🔒 对于内容丰富的文件，自动跳过，保护你的数据
- 📊 完成后显示详细的初始化报告

**强制模式**：
```
强制初始化项目结构，跳过所有确认
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

---

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

---

## 📖 使用指南

### 日常开发流程

1. **启动 Session**: 自动显示当前任务摘要
2. **开始任务**: 创建任务卡片到 `backlog/`
3. **完成任务**: 移动任务卡片到 `done/YYYY-MM/`
4. **查看日报**: 每日首次启动自动生成

### 手动命令

```markdown
# 初始化或修复项目结构
/setup-task-pilot              # 智能模式（自动判断）
/setup-task-pilot auto         # 自动模式（少询问）
/setup-task-pilot interactive  # 交互模式（详细询问）

# 推荐下一个任务
说："推荐下一个任务" 或 "接下来做什么"
→ 调用: task-suggester agent

# 生成日报
说："生成今日日报" 或 "今日进度"
→ 调用: daily-reporter agent
```

#### /setup-task-pilot 命令详解

**用途**：为中途安装插件的项目初始化文档结构，或修复损坏的项目结构

**行为**：
1. 检查现有目录和文件
2. 创建缺失的部分（目录和文件）
3. 对于已存在的文件：
   - 内容 < 5 行 → 询问是否覆盖（可能是初始化失败）
   - 内容 ≥ 5 行 → 跳过（保护现有内容）
4. **询问是否导入当前计划**到 session.md
5. **自动更新 CLAUDE.md**，添加插件使用说明
6. 完成后显示详细报告

**新增功能**：
- ✨ **智能计划导入**：创建文件后询问是否将当前开发计划写入 session.md
- ✨ **自动配置 CLAUDE.md**：在项目文档中添加 Claude Task Pilot 使用说明
- 📝 确保 AI 在后续 session 中了解项目任务管理方式

**何时使用**：
- ✅ 刚安装插件到现有项目
- ✅ 项目结构损坏需要修复
- ✅ 想要补充缺失的文件
- ❌ 不用于全新项目（会自动初始化）

---

## 🏗️ 架构

### Hooks

- **SessionStart**: 初始化检测 + 摘要显示
- **PreToolWrite**: 任务开始检测
- **PostToolWrite**: 任务完成 + 规范守护

### Agents

- **task-suggester**: 智能任务推荐
- **daily-reporter**: 日报生成

### Commands

- **/setup-task-pilot**: 项目初始化命令
  - 支持自动/交互模式
  - 智能判断文件状态
  - 保护现有数据

### Templates

- `session.md`: Session 状态模板
- `roadmap.md`: 路线图模板
- `current-sprint.md`: 冲刺模板
- `archive-index.md`: 归档索引模板
- `task-complete.md`: 完成报告模板
- `CLAUDE.md.template`: CLAUDE.md 模板（包含插件使用说明）

---

## 🎨 设计原则

1. **AI 优先**: 文档结构化、高信息密度
2. **快速启动**: 新 Session 5-10 秒了解状态
3. **任务驱动**: 所有开发活动围绕任务卡片
4. **完整追溯**: 任务生命周期记录完整
5. **非侵入式**: Hooks 观察，不干扰用户操作

---

## 🔧 故障排查

### 项目结构损坏或不完整

**症状**：某些文件或目录丢失，插件无法正常工作

**解决方案 1：使用 Skill 修复（推荐）**

```
帮我修复项目结构
```

或强制模式：

```
强制修复项目结构
```

**解决方案 2：手动重建**

```bash
# 删除损坏的文件
rm docs/session.md

# 重启 Claude Code，自动重建
claude-code
```

**解决方案 3：从模板复制**

```bash
# 从插件模板复制
cp ~/.claude/plugins/claude-task-pilot/templates/session.md docs/session.md
cp ~/.claude/plugins/claude-task-pilot/templates/roadmap.md docs/todo/roadmap.md
```

### 归档索引损坏

```bash
# 从模板重建
cp ~/.claude/plugins/claude-task-pilot/templates/archive-index.md \
   docs/done/archive-index.md
```

### Hook 未触发

```bash
# 检查插件目录
ls ~/.claude/plugins/claude-task-pilot

# 检查 plugin.json 语法
cat ~/.claude/plugins/claude-task-pilot/.claude-plugin/plugin.json | jq
```

### Agent 调用失败

- 检查 Agent 文件是否存在
- 检查 `plugin.json` 中的 agent 声明
- 查看 Claude Code 日志

---

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
- [ ] 中途安装插件后 /setup-task-pilot 命令正常工作
- [ ] 智能判断功能：小文件询问，大文件跳过
- [ ] auto/interactive 模式正常切换
- [ ] 任务创建后 session.md 自动更新
- [ ] 任务完成后自动生成报告
- [ ] 归档索引自动更新
- [ ] 智能推荐功能正常
- [ ] session.md 行数限制生效

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

### 开发指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature-amazing-feature`)
5. 开启 Pull Request

---

## 📄 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

---

## 🔗 相关链接

- **GitHub 仓库**: https://github.com/youtao/claude-task-pilot
- **问题反馈**: https://github.com/youtao/claude-task-pilot/issues
- **发布历史**: https://github.com/youtao/claude-task-pilot/releases

---

**最后更新**: 2026-01-30
**版本**: v1.3.0
**维护者**: youtao

---

## 📜 更新日志

### v1.4.0 (2026-01-30)

**新增功能**：
- ✨ 初始化后询问是否导入当前计划到 session.md
- ✨ 自动在 CLAUDE.md 中添加插件使用说明
- 📝 创建 CLAUDE.md.template 模板文件

**改进内容**：
1. **智能计划导入**：
   - 创建模板文件后询问用户
   - 引导提供：当前任务、已完成工作、下一步计划、问题 blockers
   - 将计划写入 session.md 保持连续性

2. **自动配置 CLAUDE.md**：
   - 检测并创建/更新 CLAUDE.md
   - 添加简洁的插件使用说明
   - 让 AI 在新 session 中了解项目使用 Claude Task Pilot

3. **简化模板**：
   - CLAUDE.md.template 只包含核心信息
   - 明确指引 AI 查看 docs/session.md

**用户体验提升**：
- ✅ 新项目初始化后即可导入当前计划
- ✅ AI 在后续 session 中自动了解项目背景
- ✅ 保持任务追踪的连续性

### v1.3.0 (2026-01-30)

**新增功能**：
- ✨ 添加 `/setup-task-pilot` 斜杠命令，支持直接初始化
- 🎯 支持 `auto` 和 `interactive` 两种模式
- 📖 更新 README 文档，添加命令使用说明

**命令参数**：
- 无参数：智能模式（自动判断文件状态）
- `auto`：自动模式（智能判断，少询问）
- `interactive`：交互模式（详细询问每个文件）

### v1.2.1 (2026-01-30)

**测试版本**：
- 🔧 版本管理流程验证

### v1.2.0 (2026-01-30)

**改进**：
- 📖 在 CLAUDE.md 添加版本管理章节
- 📦 建立版本发布流程和检查清单
- 🔧 规范化版本号同步机制

**文档完善**：
- 版本管理最佳实践
- 完整的发布流程
- 发布前检查清单

### v1.1.0 (2026-01-30)

**新增功能**：
- ✨ 添加 setup-task-pilot 命令，支持中途安装插件的项目初始化
- 🤖 智能判断机制：根据文件内容自动决定是否重建
- 🔒 数据保护：从不删除已有内容，只创建或经确认后覆盖
- 📊 详细的初始化报告

**改进**：
- 📖 更新文档，添加命令使用说明
- 🧪 添加测试场景和清单

### v1.0.0 (2026-01-30)

**初始版本**：
- ✅ 自动初始化
- ✅ 任务状态追踪
- ✅ 智能任务推荐
- ✅ 日报生成
- ✅ 规范守护
