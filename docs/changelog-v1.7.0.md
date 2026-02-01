# 更新日志 v1.7.0

**发布日期**: 2026-02-01

---

## 🎯 核心功能：完全自动的文档同步

### 用户体验问题

**用户反馈**: "能做到自动吗？"

用户不想每次手动运行 `/sync-progress` 命令，希望插件能够自动检测并同步文档。

### 解决方案

新增 **自动同步 Hook**，在文件操作后自动检测并修复文档不一致，无需手动干预。

---

## ✨ 新增功能

### 🤖 自动文档同步

**完全自动**的文档同步，无需手动运行命令。

#### 触发时机

自动同步在以下情况触发：

1. **编辑任务卡片**
   - 在 `backlog/task-*.md` 中添加完成标记
   - 自动归档到 `done/YYYY-MM/`

2. **编辑关键文档**
   - 更新 `session.md`
   - 更新 `current-sprint.md`
   - 更新 `roadmap.md`

3. **执行文件操作**
   - 使用 Bash 命令移动/删除文件
   - 自动更新相关索引

#### 三种同步模式

**1. prompt 模式（推荐，默认）**

```yaml
auto_sync_mode: "prompt"
```

- 检测到问题后询问用户
- 显示发现的问题详情
- 用户确认后才执行修复

**示例**:
```markdown
🔄 检测到文档不一致，自动同步中...

发现以下问题:
1. [TASK_COMPLETED_NOT_ARCHIVED] task-001-feature.md
   - 任务已完成但未归档

是否自动修复？ (Y/n) y

✅ 已归档: task-001-feature.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md

✅ 自动同步完成
```

**2. auto 模式（完全自动）**

```yaml
auto_sync_mode: "auto"
```

- 检测到问题后自动修复
- 不询问用户
- 显示简要结果

**示例**:
```markdown
🔄 检测到文档不一致，自动同步中...

✅ 已归档: task-001-feature.md
✅ 已更新 session.md
✅ 已更新 current-sprint.md

✅ 自动同步完成
```

**3. silent 模式（静默）**

```yaml
auto_sync_mode: "silent"
```

- 只修复明显的问题
- 不显示任何输出
- 用户无感知

**适用场景**: 不想被打扰的高级用户

#### 配置选项

```yaml
---
# 自动文档同步
auto_sync: true                   # 启用自动同步（默认）
auto_sync_mode: "prompt"          # 同步模式
                                  # - auto: 完全自动，不询问
                                  # - prompt: 提示用户（推荐）
                                  # - silent: 静默修复
auto_sync_threshold: 3            # 问题数量阈值
auto_sync_exclude:                # 排除的文件/目录
  - "node_modules/"
  - ".git/"
---
```

---

## 🔧 技术实现

### PostToolWrite Hook 增强

新增 `hooks/auto-sync.md`，实现自动检测和同步逻辑。

#### 检测逻辑

```javascript
// 快速检测（不扫描所有文件）
const issues = []

// 检查 1: 编辑了 backlog 中的任务卡片
if (filePath.match(/todo\/backlog\/task-\d{3}-[\w-]+\.md$/)) {
  const content = await readFile(filePath, 'utf-8')

  // 检查是否添加了完成标记
  if (content.includes('**完成时间**') || content.includes('状态: ✅')) {
    issues.push({
      type: 'TASK_COMPLETED_NOT_ARCHIVED',
      file: filePath,
      message: '任务已完成但未归档',
      autoFix: true
    })
  }
}
```

#### 自动修复

```javascript
// 执行轻量级同步
for (const issue of issues) {
  if (issue.type === 'TASK_COMPLETED_NOT_ARCHIVED') {
    // 自动归档
    await archiveTask(issue.file)
    await updateSessionMd()
    await updateCurrentSprint()
  }
}
```

#### 智能判断

```javascript
// 根据配置决定是否自动修复
if (config.auto_sync_mode === 'auto') {
  shouldAutoFix = true  // 完全自动
} else if (config.auto_sync_mode === 'prompt') {
  shouldAutoFix = await askUser(message)  // 询问用户
} else if (config.auto_sync_mode === 'silent') {
  shouldAutoFix = issues.every(i => i.autoFix)  // 只修复明显的
}
```

---

## 📚 新增文档

### Hook 文档
- **[hooks/auto-sync.md](../hooks/auto-sync.md)** - 自动同步 Hook 完整文档

### 配置更新
- **[README.md](../README.md)** - 添加自动同步配置说明
- **[docs/changelog-v1.7.0.md](changelog-v1.7.0.md)** - 本更新日志

---

## 🎨 用户体验改进

### 1. 零手动操作

**之前**:
```bash
# 1. 编辑任务卡片
vim docs/todo/backlog/task-001.md
# 添加完成标记

# 2. 手动运行同步命令
/sync-progress
```

**现在**:
```bash
# 1. 编辑任务卡片
vim docs/todo/backlog/task-001.md
# 添加完成标记

# 2. 保存文件，自动同步！
✅ 自动完成
```

### 2. 灵活的控制

- **想要完全自动** → `auto_sync_mode: "auto"`
- **想要确认提示** → `auto_sync_mode: "prompt"`（推荐）
- **想要无感知** → `auto_sync_mode: "silent"`
- **想要手动控制** → `auto_sync: false`

### 3. 双层保障

1. **PostToolWrite Hook** - 操作后立即同步
2. **SessionStart Hook** - 启动时检查修复

确保文档始终一致！

---

## 📖 使用指南

### 推荐配置（大多数用户）

在项目根目录创建 `.claude/claude-task-pilot.local.md`：

```yaml
---
auto_sync: true
auto_sync_mode: "prompt"
---
```

**工作流程**:
1. 正常工作，编辑任务卡片
2. 添加完成标记
3. 保存文件
4. 插件提示是否自动修复
5. 确认后自动完成

### 完全自动（高级用户）

```yaml
---
auto_sync: true
auto_sync_mode: "auto"
---
```

**工作流程**:
1. 正常工作，编辑任务卡片
2. 添加完成标记
3. 保存文件
4. 自动完成（无提示）

### 手动控制（谨慎用户）

```yaml
---
auto_sync: false
---
```

**工作流程**:
1. 正常工作，编辑任务卡片
2. 添加完成标记
3. 保存文件
4. 定期手动运行 `/sync-progress`

---

## 🆚 功能对比

### 手动 vs 自动

| 特性 | 手动命令 | 自动同步 |
|------|---------|---------|
| **触发方式** | 用户运行命令 | Hook 自动触发 |
| **同步时机** | 用户决定 | 操作后立即 |
| **需要干预** | 每次 | 可配置 |
| **检测范围** | 全面 | 轻量级 |
| **适用场景** | 定期总结 | 日常工作 |

### 推荐组合

```bash
# 日常使用 → 依赖自动同步
# 编辑任务卡片
vim docs/todo/backlog/task-001.md
# 添加完成标记
# 保存 → 自动同步 ✅

#定期总结 → 手动全面同步
/sync-progress full
```

---

## 🐛 已知限制

### 1. 不触发自动同步的操作

以下操作不会触发自动同步：

- 使用 `cp` 复制文件（不是移动）
- 使用外部编辑器修改文件（如果 Hook 未捕获）
- 通配符移动（`mv *.md done/`）

**解决方案**: 手动运行 `/sync-progress`

### 2. 配置优先级

项目级配置优先于全局配置：
- `.claude/claude-task-pilot.local.md`（项目）
- `~/.claude/plugins/claude-task-pilot/.claude/claude-task-pilot.local.md`（全局）

### 3. 性能影响

自动同步使用轻量级检测，对性能影响极小（< 100ms）。

---

## 🚀 性能优化

### 1. 轻量级检测

- 不扫描所有文件
- 只检查刚操作的文件
- 使用快速路径匹配

### 2. 智能触发

- 不是所有文件操作都触发
- 只在检测到特定模式时触发
- 避免不必要的同步

### 3. 延迟执行

- 避免频繁触发
- 1 秒内多次操作只触发一次
- 使用防抖机制

---

## 🔮 未来计划

### v1.8.0 计划

- [ ] 增量同步（只同步变更部分）
- [ ] 同步历史记录
- [ ] 回滚功能（恢复到之前状态）
- [ ] 同步统计和可视化

### v2.0.0 愿景

- [ ] 实时文档同步（WebSocket）
- [ ] 多人协作支持
- [ ] 冲突自动解决
- [ ] 云端备份集成

---

## ⚠️ 升级注意事项

### 向后兼容性

✅ **完全向后兼容**
- 现有命令不受影响
- 自动同步默认启用
- 可随时禁用
- 配置格式无变化

### 推荐升级步骤

```bash
# 1. 拉取最新代码
cd ~/.claude/plugins/claude-task-pilot
git pull origin master

# 2. 配置自动同步（可选）
# 在项目中创建 .claude/claude-task-pilot.local.md
cat > .claude/claude-task-pilot.local.md << 'EOF'
---
auto_sync: true
auto_sync_mode: "prompt"
---
EOF

# 3. 测试自动同步
# 编辑一个任务卡片，添加完成标记
vim docs/todo/backlog/task-001.md
# 保存，观察是否触发自动同步
```

---

## 📊 版本历史

### v1.6.0 → v1.7.0

| 功能 | v1.6.0 | v1.7.0 |
|------|--------|--------|
| 手动同步 | ✅ | ✅ |
| 自动同步 | ❌ | ✅ |
| 同步模式 | 无 | 3 种 |
| 配置选项 | 基础 | 丰富 |
| 用户体验 | 需手动操作 | 零操作 |

---

## 🙏 致谢

感谢用户反馈，帮助我们实现"完全自动"的文档同步！

---

## 📞 反馈与支持

- **GitHub Issues**: [提交问题](https://github.com/youtao/claude-task-pilot/issues)
- **讨论区**: [GitHub Discussions](https://github.com/youtao/claude-task-pilot/discussions)

---

**下载**: [v1.7.0 Release](https://github.com/youtao/claude-task-pilot/releases/tag/v1.7.0)

**完整文档**: [README.md](../README.md)
