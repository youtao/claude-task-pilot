# Claude Task Pilot

**AI 驱动的 Claude Code 任务管理插件**

[![版本](https://img.shields.io/badge/版本-2.0.0-blue.svg)](https://github.com/youtao/claude-task-pilot)
[![许可证](https://img.shields.io/badge/许可证-MIT-green.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-插件-purple.svg)](https://github.com/anthropics/claude-code)

---

## 🎯 为什么选择 Claude Task Pilot？

Claude Task Pilot 是专为 Claude Code 工作流设计的 AI 原生任务管理系统：

- **🤖 AI 思维模式**: 文档结构专为 LLM 理解优化
- **⚡ 手动触发**: 通过命令精确控制操作时机，更稳定
- **🎯 智能推荐**: 基于优先级和依赖关系的任务建议
- **🚀 快速启动**: 5-10 秒了解项目状态
- **🔄 完整追踪**: 从设计到归档的完整生命周期管理

---

## ✨ 核心功能

### 1. 文档初始化
- ✅ 智能检测项目状态
- ✅ 创建符合规范的文档结构
- ✅ 生成模板文件

### 2. 文档同步
- ✅ **快速同步**: 日常使用的快速模式
- ✅ **完全同步**: 包含 roadmap.md 的全面同步
- ✅ **验证模式**: 检查不一致，不修改文件

### 3. 任务管理

#### 完成任务
```bash
/sync-progress --complete-task task-001
```

**自动执行**:
- ✅ 归档任务到 `docs/done/YYYY-MM/`
- ✅ 更新 session.md、current-sprint.md
- ✅ 更新 archive-index.md
- ✅ 归档关联的设计文档
- ✅ 推荐下一个任务

#### 转换设计文档
```bash
/sync-progress --convert-design
```

**智能生成**:
- 按模块拆分任务（backend/frontend/testing/doc）
- 多维度优先级评分
- 自动识别依赖关系

#### 同步进度
```bash
/sync-progress          # 快速同步（日常使用）
/sync-progress full     # 完全同步（每周总结）
/sync-progress verify   # 验证模式（检查不一致）
```

### 4. 智能功能
- **🎯 任务推荐**: 基于优先级和依赖关系推荐下一个任务
- **📊 日报生成**: 自动生成每日进度报告
- **🔄 规范守护**: 确保 session.md 不超过 80 行

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
└── done/                   # 已完成任务和设计归档
    ├── archive-index.md    # 归档索引
    └── YYYY-MM/            # 按月归档
        ├── task-XXX-*.md   # 已完成任务
        └── *-design.md     # 已完成设计
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

### 初始化项目

```bash
/setup-task-pilot
```

将会创建:
- docs/session.md
- docs/todo/roadmap.md
- docs/todo/current-sprint.md
- docs/done/archive-index.md
- docs/todo/backlog/
- docs/done/YYYY-MM/

### 配置（可选）

在项目根目录创建 `.claude/claude-task-pilot.local.md`：

```yaml
---
docs_root: "docs"
session_max_lines: 80
---
```

---

## 💡 日常使用

### 推荐工作流程

```bash
# 1. 查看当前任务
cat docs/session.md

# 2. Claude Code 完成工作
"请实现用户登录功能"

# 3. 标记任务完成
/sync-progress --complete-task task-001

# 4. 查看推荐的下一个任务
[自动显示任务推荐]
```

### 定期同步

```bash
# 日常工作后
/sync-progress

# 每周或长时间工作后
/sync-progress full

# 检查数据一致性
/sync-progress verify
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
---
```

### 全局配置

编辑 `~/.claude/plugins/claude-task-pilot/.claude/claude-task-pilot.local.md`

**优先级**: 项目级配置 > 全局配置 > 默认配置

---

## 🏗️ 架构

### Commands

- **/setup-task-pilot**: 项目初始化
- **/sync-progress**: 文档同步 + 完成任务 + 转换设计文档

### Agents

- **task-suggester**: 智能任务推荐
- **daily-reporter**: 日报生成
- **design-to-tasks**: 设计文档转换

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
2. **手动触发**: 精确控制操作时机
3. **快速启动**: 新 Session 5-10 秒了解状态
4. **任务驱动**: 所有开发活动围绕任务卡片
5. **完整追溯**: 任务生命周期记录完整

---

## 🔧 故障排查

### Linux 安装错误: EXDEV cross-device link not permitted

**错误信息**:
```
Error: Failed to install: EXDEV: cross-device link not permitted, rename
'/home/youtao/.claude/plugins/cache/claude-task-pilot' -> '/tmp/claude-plugin-temp-xxx'
```

**问题原因**:
- Linux 的 `rename()` 系统调用无法跨文件系统工作
- `/home` 分区和 `/tmp` 分区位于不同的文件系统上

**解决方案**:

#### 方案 1: 手动克隆（推荐）

```bash
cd ~/.claude/plugins
git clone https://github.com/youtao/claude-task-pilot.git
```

#### 方案 2: 使用统一的临时目录

```bash
mkdir -p ~/.claude/tmp
export TMPDIR=~/.claude/tmp
claude-code
```

#### 方案 3: 系统级解决（需要 root 权限）

```bash
sudo mkdir -p /home/tmp
sudo chmod 1777 /home/tmp
sudo mount --bind /tmp /home/tmp
echo "/tmp /home/tmp none bind 0 0" | sudo tee -a /etc/fstab
```

### 文档不同步

**解决**:
```bash
# 1. 验证模式检查
/sync-progress verify

# 2. 手动全面同步
/sync-progress full
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

---

## 📚 详细文档

### 核心文档

- **[README.md](README.md)** - 本文档
- **[CLAUDE.md](CLAUDE.md)** - 项目 AI 指导文档

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

### v2.0.0 (2026-02-05) - 精简优化 ⭐

**改进内容**:
- ✨ 删除冗余文档目录（docs/）
- ✨ 精简 CLAUDE.md（从 783 行 → 292 行）
- ✨ 精简 README.md（从 674 行 → 约 300 行）
- ✨ 优化文档结构，提高可维护性

**删除内容**:
- docs/changelog-*.md（已合并到 README）
- docs/configuration-guide.md（配置说明已整合）
- docs/task-completion-workflow.md（使用说明已整合）
- docs/test-bash-mv-detection.md（测试文档）

### v1.8.0 (2026-02-01) - 设计文档自动归档

**新增功能**:
- ✨ **设计文档自动归档** - 任务完成后自动归档关联的设计文档
- ✨ 与任务卡片一起归档到 `docs/done/YYYY-MM/`
- ✨ 添加完成状态和关联任务信息

### v1.7.0 (2026-02-01) - 完全自动的文档同步

**新增功能**:
- ✨ **自动文档同步 Hook** - 完全自动，无需手动命令
- ✨ 三种同步模式: prompt（推荐）/ auto（完全自动）/ silent（静默）
- ✨ 智能检测文档不一致并自动修复

### v1.6.0 (2026-02-01) - 文档同步命令

**新增功能**:
- ✨ **/sync-progress 命令** - 全面同步所有项目文档
- ✨ 三种同步模式: quick/full/verify
- ✨ 智能问题检测: 未归档任务、状态不一致、缺失索引

### v1.5.2 (2026-02-01) - 任务完成命令

**新增功能**:
- ✨ **/complete-task 命令** - 标记任务完成并更新文档
- ✨ 支持多种参数格式: 任务ID/文件路径/默认
- ✨ 自动更新所有相关文档
- ✨ 智能推荐下一个任务

### v1.5.0 (2026-02-01)

**新增功能**:
- ✨ 设计文档自动转换（design-to-tasks agent）
- ✨ 自动检测设计文档创建并生成任务卡片
- ✨ 手动转换命令 `/convert-design-to-tasks`
- ✨ 智能任务拆分（按模块、复杂度）
- ✨ 优先级自动判断算法
- ✨ 依赖关系自动识别

---

**最后更新**: 2026-02-05
**版本**: v2.0.0
**维护者**: youtao
