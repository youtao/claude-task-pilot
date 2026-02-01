# Claude Task Pilot

**AI 驱动的 Claude Code 任务管理插件**

[![版本](https://img.shields.io/badge/版本-1.7.0-blue.svg)](https://github.com/youtao/claude-task-pilot)
[![许可证](https://img.shields.io/badge/许可证-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-插件-purple.svg)](https://github.com/anthropics/claude-code)

---

## 🎯 为什么选择 Claude Task Pilot？

Claude Task Pilot 是专为 Claude Code 工作流设计的 AI 原生任务管理系统。与传统任务管理工具不同，它具有以下特点：

- **🤖 AI 思维模式**: 文档结构专为 LLM 理解优化
- **⚡ 零手动操作**: 完全自动的文档同步，无需手动命令
- **🎯 智能推荐**: 基于优先级和依赖关系的任务建议
- **🚀 快速启动**: 5-10 秒了解项目状态
- **🔄 完整追踪**: 从设计到归档的完整生命周期管理

---

## ✨ 核心功能

### 1. 自动初始化
- ✅ 检测项目是否已建立任务管理系统
- ✅ 自动创建符合规范的文档结构
- ✅ 生成模板文件
- ✅ 支持中途安装插件的项目

### 2. 完全自动的文档同步 ⭐ NEW
- ✅ **自动检测**: 编辑任务卡片后自动检测完成状态
- ✅ **自动归档**: 添加完成标记后自动归档任务
- ✅ **自动更新**: 同步更新所有相关文档
- ✅ **智能模式**: prompt/auto/silent 三种模式可选
- ✅ **双层保障**: 操作后同步 + 启动时检查

**工作流程**:
```bash
# 1. 编辑任务卡片
vim docs/todo/backlog/task-001.md

# 2. 添加完成标记
**完成时间**: 2026-02-01

# 3. 保存文件

# 4. 自动完成！
🔄 检测到文档不一致，自动同步中...
✅ 已归档: task-001.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md
```

### 3. 任务管理命令

#### `/complete-task` - 任务完成命令
标记任务完成并自动更新所有相关文档。

```bash
/complete-task              # 使用当前任务（从 session.md）
/complete-task task-001     # 指定任务ID
/complete-task task-001.md  # 指定文件路径
```

**自动执行**:
- ✅ 移动任务卡片到归档目录
- ✅ 创建完成报告
- ✅ 更新 session.md
- ✅ 更新 current-sprint.md
- ✅ 更新 archive-index.md
- ✅ 智能推荐下一个任务

#### `/sync-progress` - 文档同步命令
全面同步所有项目文档，确保数据一致性。

```bash
/sync-progress          # 快速同步（日常使用）
/sync-progress full     # 完全同步（每周总结）
/sync-progress verify   # 验证模式（检查不一致）
```

**解决的问题**:
- 手动保存进度时只更新部分文档
- roadmap.md、done/、backlog/ 未同步
- 文档状态不一致

#### `/convert-design-to-tasks` - 设计文档转换
将设计文档自动转换为可执行的任务卡片。

```bash
/convert-design-to-tasks                                    # 使用最新设计文档
/convert-design-to-tasks docs/plans/feature-design.md      # 指定文档
```

**智能生成**:
- 按模块拆分任务（backend/frontend/testing/doc）
- 多维度优先级评分（复杂度、依赖、模块关键性、用户价值）
- 自动识别依赖关系（数据模型→API→前端→测试）

#### `/setup-task-pilot` - 项目初始化
为中途安装插件的项目初始化文档结构。

```bash
/setup-task-pilot              # 智能模式（自动判断）
/setup-task-pilot auto         # 自动模式（少询问）
/setup-task-pilot interactive  # 交互模式（详细询问）
```

### 4. 智能功能
- **🎯 任务推荐**: 基于优先级和依赖关系推荐下一个任务
- **📊 日报生成**: 自动生成每日进度报告
- **🔄 规范守护**: 确保 session.md 不超过 80 行
- **🔍 一致性检查**: SessionStart 时自动检测并修复不一致

### 5. 设计文档转换
- **🤖 自动触发**: 检测设计文档创建，自动生成任务卡片
- **🧠 智能拆分**: 按模块、复杂度自动拆分任务
- **📊 优先级判断**: 多维度算法自动判断任务优先级
- **🔗 依赖识别**: 自动识别任务间依赖关系

---

## 📁 文档结构

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

**创建时间**: 2026-02-01
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

### 配置自动同步（推荐）

在项目根目录创建 `.claude/claude-task-pilot.local.md`：

```yaml
---
# 自动文档同步（NEW）
auto_sync: true                   # 启用自动同步（默认）
auto_sync_mode: "prompt"          # 同步模式
                                  # - auto: 完全自动，不询问
                                  # - prompt: 提示用户（推荐）⭐
                                  # - silent: 静默修复
---
```

**详细配置指南**: [docs/configuration-guide.md](docs/configuration-guide.md)

---

## 💡 日常使用

### 推荐工作流程（完全自动）

```bash
# 1. 查看当前任务
cat docs/session.md

# 2. Claude Code 完成工作
"请实现用户登录功能"

# 3. 编辑任务卡片，添加完成标记
vim docs/todo/backlog/task-001.md
# 添加: **完成时间**: 2026-02-01

# 4. 保存文件

# 5. 自动同步完成！
🔄 检测到文档不一致，自动同步中...
✅ 已归档: task-001.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md
✅ 自动同步完成
```

### 手动命令工作流程

```bash
# 1. 查看当前任务
cat docs/session.md

# 2. Claude Code 完成工作

# 3. 标记任务完成
/complete-task

# 4. 查看推荐
[自动显示下一个任务推荐]
```

### 定期全面同步

```bash
# 每周或长时间工作后
/sync-progress full
```

---

## 📖 详细使用指南

### 1. 创建任务

在 `docs/todo/backlog/` 创建新任务卡片：

```markdown
# task-001: 用户登录功能

**创建时间**: 2026-02-01
**优先级**: P0
**模块**: backend

## 任务描述
实现用户登录功能，支持邮箱和密码登录。

## 验收标准
- [ ] 登录 API: POST /api/auth/login
- [ ] 密码加密存储
- [ ] JWT Token 生成
- [ ] 单元测试覆盖率 > 80%
```

**自动触发**:
- ✅ 更新 `session.md` "当前任务"
- ✅ 更新 `current-sprint.md` 状态为 🔄

### 2. 完成任务

#### 方式 A: 自动同步（推荐）⭐

```bash
# 1. 编辑任务卡片
vim docs/todo/backlog/task-001.md

# 2. 添加完成标记
**完成时间**: 2026-02-01

# 3. 保存文件，自动完成！
```

#### 方式 B: 使用命令

```bash
/complete-task task-001
```

#### 方式 C: 手动移动

```bash
mv docs/todo/backlog/task-001.md docs/done/2026-02/
```

**所有方式都会**:
- ✅ 更新 `session.md` "上一个任务"
- ✅ 更新 `current-sprint.md` 状态为 ✅
- ✅ 更新 `archive-index.md`
- ✅ 智能推荐下一个任务

### 3. 同步文档

#### 何时需要同步

- ✅ 手动保存进度后发现文档不一致
- ✅ 长时间工作后
- ✅ 怀疑文档有问题时

#### 快速同步

```bash
/sync-progress
```

#### 完全同步

```bash
/sync-progress full
```

#### 验证模式

```bash
/sync-progress verify
```

### 4. 转换设计文档

#### 自动转换（推荐）

```bash
# 1. 使用 brainstorming
/brainstorm

# 2. 设计文档创建
# docs/plans/2026-02-01-user-authentication-design.md

# 3. 自动生成任务卡片
📋 正在分析设计文档...
✅ 已创建 5 个任务卡片
```

#### 手动转换

```bash
/convert-design-to-tasks
```

---

## 🔧 配置

### 项目级配置

在项目根目录创建 `.claude/claude-task-pilot.local.md`：

```yaml
---
# 基础配置
docs_root: "docs"
session_max_lines: 80
auto_recommend: true
daily_report: true

# 设计文档自动转换
auto_convert_designs: true        # 启用自动转换
prompt_before_convert: false      # 不询问，直接生成

# 自动文档同步 ⭐ NEW
auto_sync: true                   # 启用自动同步（默认）
auto_sync_mode: "prompt"          # 同步模式
                                  # - auto: 完全自动，不询问
                                  # - prompt: 提示用户（推荐）
                                  # - silent: 静默修复
auto_sync_threshold: 3            # 问题数量阈值
auto_sync_exclude:                # 排除的目录
  - "node_modules/"
  - ".git/"
---
```

### 全局配置

编辑 `~/.claude/plugins/claude-task-pilot/.claude/claude-task-pilot.local.md`

**优先级**: 项目级配置 > 全局配置 > 默认配置

---

## 🏗️ 架构

### Hooks

- **SessionStart**: 初始化检测 + 摘要显示 + 未处理任务检测
- **PreToolWrite**: 任务开始检测
- **PostToolWrite**: 任务完成 + Bash 命令检测 + 设计文档转换
- **Auto-Sync**: 自动文档同步 ⭐ NEW

### Agents

- **task-suggester**: 智能任务推荐
- **daily-reporter**: 日报生成
- **design-to-tasks**: 设计文档转换

### Commands

- **/complete-task**: 标记任务完成 ⭐
- **/sync-progress**: 文档同步 ⭐
- **/convert-design-to-tasks**: 设计文档转换
- **/setup-task-pilot**: 项目初始化

### Templates

- `session.md`: Session 状态模板
- `roadmap.md`: 路线图模板
- `current-sprint.md`: 冲刺模板
- `archive-index.md`: 归档索引模板
- `task-complete.md`: 完成报告模板
- `task-backlog.md`: 任务卡片模板

---

## 🎨 设计原则

1. **AI 优先**: 文档结构化、高信息密度
2. **零手动操作**: 完全自动的文档同步
3. **快速启动**: 新 Session 5-10 秒了解状态
4. **任务驱动**: 所有开发活动围绕任务卡片
5. **完整追溯**: 任务生命周期记录完整
6. **非侵入式**: Hooks 观察，不干扰用户操作

---

## 🆚 功能对比

### 手动 vs 自动

| 操作 | 手动方式 | 自动方式 ⭐ |
|------|---------|------------|
| 完成任务 | /complete-task | 编辑任务卡片 |
| 同步文档 | /sync-progress | 自动检测 |
| 归档任务 | mv 命令 | 自动归档 |
| 更新文档 | 手动更新 | 自动更新 |

### 推荐组合

```bash
# 日常使用 → 完全自动（auto_sync_mode: "auto"）
# 编辑任务卡片 → 自动完成

#定期总结 → 手动全面同步
/sync-progress full
```

---

## 🔧 故障排查

### 自动同步不触发

**检查**:
```bash
# 1. 确认配置
cat .claude/claude-task-pilot.local.md | grep auto_sync

# 2. 应该看到: auto_sync: true

# 3. 重启 Claude Code
```

### 文档不同步

**解决**:
```bash
# 1. 验证模式检查
/sync-progress verify

# 2. 手动全面同步
/sync-progress full
```

### Hook 未触发

**检查**:
```bash
# 检查插件目录
ls ~/.claude/plugins/claude-task-pilot/hooks/

# 应该看到:
# - auto-sync.md
# - session-start.md
# - post-tool-write.md
# - pre-tool-write.md
```

---

## 🧪 测试

### 快速测试

```bash
# 1. 创建测试项目
mkdir ~/test-pilot && cd ~/test-pilot

# 2. 启动 Claude Code
claude-code

# 3. 创建任务
cat > docs/todo/backlog/task-001.md << 'EOF'
# task-001: 测试
**创建时间**: 2026-02-01
**完成时间**: 2026-02-01
EOF

# 4. 观察是否自动归档
```

### 测试清单

- [ ] 新项目自动初始化
- [ ] 自动同步功能正常
- [ ] 任务创建后自动更新 session.md
- [ ] 任务完成后自动归档
- [ ] 所有文档自动同步
- [ ] 智能推荐功能正常
- [ ] session.md 行数限制生效

---

## 📚 详细文档

### 核心文档

- **[README.md](README.md)** - 本文档
- **[CLAUDE.md](CLAUDE.md)** - 项目 AI 指导文档
- **[docs/configuration-guide.md](docs/configuration-guide.md)** - 完整配置指南

### 功能文档

- **[commands/complete-task.md](commands/complete-task.md)** - 任务完成命令
- **[commands/sync-progress.md](commands/sync-progress.md)** - 文档同步命令
- **[commands/convert-design-to-tasks.md](commands/convert-design-to-tasks.md)** - 设计文档转换
- **[hooks/auto-sync.md](hooks/auto-sync.md)** - 自动同步 Hook

### 更新日志

- **[docs/changelog-v1.7.0.md](docs/changelog-v1.7.0.md)** - v1.7.0 更新日志
- **[docs/changelog-v1.6.0.md](docs/changelog-v1.6.0.md)** - v1.6.0 更新日志
- **[docs/changelog-v1.5.2.md](docs/changelog-v1.5.2.md)** - v1.5.2 更新日志

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

### 开发指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
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

## 📜 更新日志

### v1.7.0 (2026-02-01) - 完全自动的文档同步 ⭐

**新增功能**:
- ✨ **自动文档同步 Hook** - 完全自动，无需手动命令
- ✨ 三种同步模式: prompt（推荐）/ auto（完全自动）/ silent（静默）
- ✨ 智能检测文档不一致并自动修复
- ✨ 双层保障: 操作后同步 + 启动时检查

**改进内容**:
1. **零手动操作**:
   - 编辑任务卡片，添加完成标记
   - 保存文件，自动完成
   - 无需运行任何命令

2. **灵活配置**:
   - `auto_sync`: 启用/禁用自动同步
   - `auto_sync_mode`: 选择同步模式
   - `auto_sync_threshold`: 问题数量阈值

3. **完整文档**:
   - [docs/configuration-guide.md](docs/configuration-guide.md) - 详细配置指南
   - [hooks/auto-sync.md](hooks/auto-sync.md) - Hook 实现文档

**使用方式**:
```yaml
# .claude/claude-task-pilot.local.md
auto_sync: true
auto_sync_mode: "prompt"
```

### v1.6.0 (2026-02-01) - 文档同步命令

**新增功能**:
- ✨ **/sync-progress 命令** - 全面同步所有项目文档
- ✨ 三种同步模式: quick/full/verify
- ✨ 智能问题检测: 未归档任务、状态不一致、缺失索引
- ✨ 详细的同步报告

**解决问题**:
- 手动保存进度时只更新部分文档
- roadmap.md、done/、backlog/ 未同步
- 文档状态不一致

### v1.5.2 (2026-02-01) - 任务完成命令

**新增功能**:
- ✨ **/complete-task 命令** - 标记任务完成并更新文档
- ✨ 支持多种参数格式: 任务ID/文件路径/默认
- ✨ 自动更新所有相关文档
- ✨ 智能推荐下一个任务

**改进内容**:
1. **简化工作流程**:
   - 不需要手动移动文件
   - 不需要手动更新多个文档
   - 一个命令完成所有操作

2. **完整更新**:
   - 移动任务卡片到归档目录
   - 更新 session.md
   - 更新 current-sprint.md
   - 更新 archive-index.md
   - 推荐下一个任务

### v1.5.0 (2026-02-01)

**新增功能**:
- ✨ 设计文档自动转换（design-to-tasks agent）
- ✨ 自动检测设计文档创建并生成任务卡片
- ✨ 手动转换命令 `/convert-design-to-tasks`
- ✨ 智能任务拆分（按模块、复杂度）
- ✨ 优先级自动判断算法
- ✨ 依赖关系自动识别

---

**最后更新**: 2026-02-01
**版本**: v1.7.0
**维护者**: youtao
