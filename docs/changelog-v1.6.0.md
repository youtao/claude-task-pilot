# 更新日志 v1.6.0

**发布日期**: 2026-02-01

---

## 🎯 核心问题解决

### 问题描述

用户反馈：手动要求 AI 保存进度时，Claude Code 只同步了 `session.md` 和 `current-sprint.md`，其他关键文档（`roadmap.md`、`done/`、`backlog/`）没有进行相关更新。

### 解决方案

新增 `/sync-progress` 命令，提供全面的文档同步功能，确保所有项目文档保持一致。

---

## ✨ 新增功能

### 🔄 文档同步命令 (`/sync-progress`)

全新的文档同步命令，解决"手动保存进度时文档同步不完整"的问题。

#### 三种同步模式

**1. 快速同步（默认）**
```bash
/sync-progress
# 或
/sync-progress quick
```

**功能**:
- ✅ 更新 `session.md` 和 `current-sprint.md`
- ✅ 检测并归档未归档的完成任务
- ✅ 修复状态不一致问题
- ✅ 生成简洁的同步报告

**适用场景**:
- 日常工作中的快速同步
- 完成 1-2 个任务后
- 每日工作结束

**2. 完全同步**
```bash
/sync-progress full
```

**额外功能**:
- 所有快速同步操作
- ✅ 更新 `roadmap.md` 进度
- ✅ 扫描所有任务卡片，更新状态
- ✅ 重新生成 `archive-index.md`（如果损坏）
- ✅ 详细的同步报告和进度统计

**适用场景**:
- 阶段性总结（每周、每月）
- 长时间工作后
- 感觉文档不一致时

**3. 验证模式**
```bash
/sync-progress verify
```

**功能**:
- 🔍 不修改任何文件
- 🔍 只检查数据一致性
- 🔍 报告发现的问题
- 🔍 提供修复建议

**适用场景**:
- 怀疑文档有问题时
- 定期检查（每周）
- 同步前的预检查

#### 智能问题检测

命令会自动检测以下问题：

1. **未归档的完成任务**
   - 检测 `backlog/` 中已标记完成的任务
   - 自动归档到 `done/YYYY-MM/`

2. **状态不一致**
   - `current-sprint.md` 标记为完成，但任务仍在 `backlog/`
   - `current-sprint.md` 标记为进行中，但任务已在 `done/`

3. **缺失的归档记录**
   - 任务已归档，但 `archive-index.md` 中没有记录
   - 自动补充归档索引

4. **文件冲突**
   - 同一任务同时存在于 `backlog/` 和 `done/`
   - 提示用户手动解决

#### 同步报告示例

```markdown
============================================================
✅ 同步完成！
============================================================

📊 同步摘要
修复问题: 3 个
- 归档未归档任务: 1 个
- 修复状态不一致: 2 个
- 补充归档索引: 0 个

当前状态:
- backlog/: 4 个任务
- done/: 11 个已完成任务
- 总进度: 73%

下一步:
1. 查看当前任务: cat docs/session.md
2. 查看冲刺进度: cat docs/todo/current-sprint.md
3. 继续任务: cat docs/todo/backlog/task-006.md
```

---

## 📚 新增文档

### 命令文档
- **[commands/sync-progress.md](../commands/sync-progress.md)** - 文档同步命令完整文档

### 文档更新
- **[README.md](../README.md)** - 添加 `/sync-progress` 命令说明
- **[docs/changelog-v1.6.0.md](changelog-v1.6.0.md)** - 本更新日志

---

## 🎨 用户体验改进

### 1. 更清晰的反馈

命令提供详细的执行步骤和反馈：
- 📊 显示项目状态概览
- 🔍 列出检测到的问题
- ✅ 实时显示同步进度
- 📝 生成完整的同步报告

### 2. 灵活的同步模式

根据不同场景选择合适的模式：
- 日常使用 → 快速同步
- 定期总结 → 完全同步
- 预防检查 → 验证模式

### 3. 安全的验证模式

担心同步会出错？先使用验证模式：
```bash
/sync-progress verify
```

只检查不修改，确认问题后再决定是否修复。

---

## 🔧 技术改进

### 1. 全面的文档扫描

```javascript
// 扫描所有任务相关文件
const backlogFiles = await glob('docs/todo/backlog/task-*.md')
const doneFiles = await glob('docs/done/**/task-*.md')
const sprintContent = await readFile('docs/todo/current-sprint.md', 'utf-8')
const sessionContent = await readFile('docs/session.md', 'utf-8')
```

### 2. 智能状态检测

```javascript
// 检测未归档的完成任务
if (content.includes('**完成时间**') || content.includes('状态: ✅')) {
  issues.push({
    type: 'UNARCHIVED_TASK',
    file,
    message: '任务已完成但未归档'
  })
}
```

### 3. 数据一致性验证

```javascript
// 检查 current-sprint.md 与实际文件状态是否一致
if (task.status === '✅ 完成' && isInBacklog) {
  issues.push({
    type: 'STATUS_MISMATCH',
    message: 'current-sprint.md 标记为完成，但任务仍在 backlog/'
  })
}
```

---

## 📖 使用指南

### 推荐工作流程

#### 日常工作流

```bash
# 1. 开始工作
cat docs/session.md

# 2. 完成任务后
/complete-task

# 3. 工作结束时
/sync-progress
```

#### 每周总结

```bash
# 周五下午或下周一早上
/sync-progress full

# 查看详细报告
cat docs/session.md
cat docs/todo/current-sprint.md
cat docs/todo/roadmap.md
```

#### 怀疑问题时

```bash
# 1. 先验证，不修改
/sync-progress verify

# 2. 确认有问题后，修复
/sync-progress full
```

### 自动化建议

#### Git 提交钩子

在 `.git/hooks/pre-commit` 中添加：
```bash
#!/bin/bash
# 提交前快速同步
claude-code /sync-progress quick
```

#### 定时任务

使用 cron 或 launchd 定期同步：
```bash
# 每天下午 6 点
0 18 * * * cd ~/project && claude-code /sync-progress
```

---

## 🆚 与其他命令的对比

| 特性 | /complete-task | /sync-progress |
|------|---------------|----------------|
| **主要用途** | 标记单个任务完成 | 全面同步所有文档 |
| **更新范围** | 部分文档 | 所有文档 |
| **归档任务** | 是（单个） | 是（批量） |
| **检测问题** | 否 | 是 |
| **推荐任务** | 是 | 否 |
| **更新 roadmap.md** | 否 | 是（full 模式） |
| **生成报告** | 简短 | 详细 |
| **验证模式** | 否 | 是 |

### 推荐组合

```bash
# 完成单个任务
/complete-task task-001

# 完成多个任务后，全面同步
/sync-progress full

# 定期检查
/sync-progress verify
```

---

## 🐛 已知限制

### 1. 通配符移动不支持

```bash
# ❌ 不支持
mv docs/todo/backlog/*.md docs/done/2026-02/

# ✅ 使用 /sync-progress 自动归档
/sync-progress
```

### 2. 复杂管道不支持

```bash
# ❌ 不支持
cat task-001.md | ... > docs/done/2026-02/task-001.md

# ✅ 使用 /complete-task
/complete-task task-001
```

### 3. 需要手动确认

所有模式都需要用户确认（除了错误场景），这是为了安全考虑。

---

## 🚀 性能优化

- **快速模式**: < 2 秒（小型项目 < 50 个任务）
- **完全模式**: < 5 秒（小型项目 < 50 个任务）
- **验证模式**: < 1 秒（不修改文件）

对于大型项目（> 100 个任务），性能可能略有下降。

---

## 🔮 未来计划

### v1.7.0 计划

- [ ] 自动同步模式（无需确认）
- [ ] 增量同步（只同步变更的文档）
- [ ] 同步历史记录
- [ ] 回滚功能（恢复到之前的同步状态）
- [ ] Web Dashboard 同步按钮

### v2.0.0 愿景

- [ ] 实时文档同步
- [ ] 多人协作支持
- [ ] 冲突自动解决
- [ ] 云端备份集成

---

## 📊 版本对比

### v1.5.2 → v1.6.0

| 功能 | v1.5.2 | v1.6.0 |
|------|--------|--------|
| 任务完成 | ✅ | ✅ |
| 文档同步 | 部分 | 全面 |
| 问题检测 | ❌ | ✅ |
| 验证模式 | ❌ | ✅ |
| roadmap.md 更新 | ❌ | ✅ |
| 同步报告 | 简单 | 详细 |

---

## ⚠️ 升级注意事项

### 向后兼容性

✅ **完全向后兼容**
- 现有命令不受影响
- 新命令为可选使用
- 文档格式无变化

### 推荐升级步骤

```bash
# 1. 拉取最新代码
cd ~/.claude/plugins/claude-task-pilot
git pull origin master

# 2. 验证安装
claude-code /sync-progress verify

# 3. 第一次完全同步
claude-code /sync-progress full
```

---

## 🙏 致谢

感谢用户反馈，帮助我们发现并解决"文档同步不完整"的问题！

---

## 📞 反馈与支持

- **GitHub Issues**: [提交问题](https://github.com/youtao/claude-task-pilot/issues)
- **讨论区**: [GitHub Discussions](https://github.com/youtao/claude-task-pilot/discussions)

---

**下载**: [v1.6.0 Release](https://github.com/youtao/claude-task-pilot/releases/tag/v1.6.0)

**完整文档**: [README.md](../README.md)
