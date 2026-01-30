# Claude Task Pilot - 插件审查报告

**审查日期**: 2026-01-30
**审查人**: Claude Code
**版本**: 1.0.0

---

## 📋 审查总结

**总体评价**: ⭐⭐⭐⭐ (4/5)
插件设计思路清晰，文档完善，但存在一些关键问题需要修复。

---

## 🔴 严重问题（必须修复）

### 1. Hook 触发时机错误
**位置**: [hooks/pre-tool-write.md](hooks/pre-tool-write.md:2)

**问题**:
```yaml
---
when: PreToolUse  # ❌ 错误
ifTool: Write
---
```

**应为**:
```yaml
---
when: PreToolWrite  # ✅ 正确
ifTool: Write
---
```

**影响**: Hook 无法触发，任务开始检测功能失效

---

### 2. 插件名称不一致
**位置**: 多处

**问题**:
- [plugin.json](.claude-plugin/plugin.json:2): `task-management-system`
- [README.md](README.md:1): `Task Management System Plugin`
- [CLAUDE.md](CLAUDE.md:1): `Claude Task Pilot`

**影响**: 混淆用户，发布到 GitHub 后名称不匹配

**修复方案**:
```diff
- "name": "task-management-system"
+ "name": "claude-task-pilot"
```

---

### 3. ~~缺少 package.json~~ ❌ **错误建议**

**原问题**: README 中提到 `npm test`，但没有 package.json

**更正**: Claude Code 插件**不需要** package.json

**原因**:
- Claude Code 插件不使用 Node.js 生态系统
- 只需要 `.claude-plugin/plugin.json`
- 不需要 `npm` 或其他包管理器

**正确做法**:
- ✅ 已删除 package.json
- ✅ 已更新 README，移除 `npm test` 引用
- ✅ 改为手动测试清单

---

## 🟡 中等问题（建议修复）

### 4. 缺少 LICENSE 文件
**影响**: 用户不清楚使用许可

**建议**: 添加 MIT License 文件

---

### 5. .gitignore 缺失
**影响**: 可能会提交不必要的文件

**建议**: 添加 .gitignore（见下一节）

---

### 6. Agent 调用方式不明确
**位置**: [hooks/post-tool-write.md](hooks/post-tool-write.md:116)

**问题**:
```markdown
#### 步骤 6: 智能推荐（F6）

**触发**: 任务完成后自动调用 `task-suggester` agent

**输出**:
```markdown
## 🎯 建议的下一个任务

[由 task-suggester agent 生成]
```
```

**问题**: 没有说明如何调用 Agent

**建议**: 添加具体的调用方式说明

```markdown
**调用方式**:
```javascript
// 在 Hook 中调用 Agent
const result = await runAgent("task-suggester", {
  context: {
    currentProject: getProjectInfo(),
    completedTasks: getCompletedTasks(),
    backlogTasks: getBacklogTasks()
  }
})
```
```

---

### 7. 文档路径硬编码
**位置**: 多处

**问题**: 文档中使用 `$DOCS_ROOT`，但没有说明如何设置

**建议**: 在所有文档开头添加配置说明

```markdown
## 配置参数

**默认值**:
- `DOCS_ROOT`: 从 `.claude/claude-task-pilot.local.md` 读取，默认 `"docs"`

**获取方式**:
```javascript
const DOCS_ROOT = getConfig("docs_root") || "docs"
```
```

---

## 🟢 轻微问题（可选修复）

### 8. 模板变量不一致
**位置**: templates/

**问题**: 不同模板使用不同的变量命名风格

**示例**:
- `session.md`: `{{CURRENT_DATE}}`
- `roadmap.md`: `{{LAST_UPDATE_DATE}}`
- `current-sprint.md`: `{{LAST_UPDATE_DATETIME}}`

**建议**: 统一命名规范
- 时间相关: `{{DATETIME}}` 或 `{{DATE}}` + `{{TIME}}`
- 任务相关: `{{TASK_ID}}`, `{{TASK_NAME}}`

---

### 9. 缺少示例项目
**影响**: 用户难以快速理解如何使用

**建议**: 在 README 中添加"快速开始"示例

```markdown
## 🚀 快速开始

### 1 分钟演示

```bash
# 创建测试项目
mkdir ~/test-project && cd ~/test-project

# 启动 Claude Code
claude-code

# 插件自动提示初始化，选择"是"

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
```

---

### 10. 缺少错误恢复机制
**位置**: 多处

**问题**: 没有说明当文件损坏时如何恢复

**建议**: 添加"故障排查"章节到 README

```markdown
## 🔧 故障排查

### session.md 损坏
```bash
# 删除损坏的文件
rm docs/session.md

# 重启 Claude Code，自动重建
claude-code
```

### 归档索引损坏
```bash
# 从模板重建
cp ~/.claude/plugins/claude-task-pilot/templates/archive-index.md \
   docs/done/archive-index.md
```
```

---

## 📊 优化建议

### 11. 添加自动化测试
**优先级**: 中

**建议**:
1. 创建 `tests/` 目录
2. 添加 Hook 触发测试
3. 添加 Agent 逻辑测试
4. 添加模板生成测试

---

### 12. 性能优化
**优先级**: 低

**建议**:
1. 缓存文件读取结果
2. 批量更新索引（而不是每次操作都更新）
3. 异步执行非关键操作

---

### 13. 添加配置验证
**优先级**: 中

**建议**:
```markdown
## 配置验证

在 SessionStart 时验证配置：

```javascript
function validateConfig() {
  const errors = []

  // 检查 docs_root 是否存在
  if (!exists(DOCS_ROOT)) {
    errors.push(`文档目录不存在: ${DOCS_ROOT}`)
  }

  // 检查 session_max_lines 是否合理
  if (SESSION_MAX_LINES < 50 || SESSION_MAX_LINES > 200) {
    errors.push(`session_max_lines 应在 50-200 之间`)
  }

  return errors
}
```
```

---

## ✅ 检查清单

### 必须修复（阻塞发布）
- [ ] 修复 PreToolWrite hook 的触发时机
- [ ] 统一插件名称为 `claude-task-pilot`
- [ ] 添加 package.json

### 建议修复（提升质量）
- [ ] 添加 LICENSE 文件
- [ ] 添加 .gitignore
- [ ] 添加 Agent 调用示例
- [ ] 添加配置说明
- [ ] 添加快速开始示例

### 可选优化（锦上添花）
- [ ] 统一模板变量命名
- [ ] 添加自动化测试
- [ ] 优化性能
- [ ] 添加配置验证

---

## 🎯 下一步行动

### 立即执行
1. 修复 [hooks/pre-tool-write.md](hooks/pre-tool-write.md:2) 的 hook 触发时机
2. 更新 [plugin.json](.claude-plugin/plugin.json:2) 中的插件名称
3. 创建 package.json

### 后续任务
1. 创建 .gitignore
2. 创建 LICENSE
3. 更新 README 添加快速开始示例
4. 添加故障排查章节

---

**审查完成时间**: 2026-01-30
**下次审查**: 修复完成后
