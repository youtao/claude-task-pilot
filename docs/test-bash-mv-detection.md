# 测试 Bash mv 命令检测

## 测试目的

验证使用 `mv` 命令移动任务卡片后，Hook 能正确检测并自动更新相关文档。

## 测试场景

### 场景 1: 使用 mv 命令移动任务卡片

**步骤**:
```bash
# 1. 创建一个测试任务卡片
cat > docs/todo/backlog/task-999-test.md << 'EOF'
# task-999: 测试 Bash mv 检测

**创建时间**: 2026-02-01
**优先级**: P1
**模块**: testing

## 任务描述
测试使用 mv 命令移动任务卡片后，Hook 是否能正确检测并更新文档。

## 验收标准
- [ ] 使用 mv 命令移动任务
- [ ] 自动更新 session.md
- [ ] 自动更新 current-sprint.md
- [ ] 自动更新 archive-index.md
EOF

# 2. 使用 mv 命令移动任务到归档目录
mv docs/todo/backlog/task-999-test.md docs/done/2026-02/

# 3. 检查文档是否自动更新
cat docs/session.md | grep "task-999"
cat docs/todo/current-sprint.md | grep "task-999"
cat docs/done/archive-index.md | grep "task-999"
```

**预期结果**:
- ✅ `session.md` 的"上一个任务"表格中包含 task-999
- ✅ `current-sprint.md` 中 task-999 状态为 ✅ 完成
- ✅ `archive-index.md` 中包含 task-999 的归档记录

### 场景 2: 使用 git mv 命令移动任务卡片

**步骤**:
```bash
# 1. 创建另一个测试任务
cat > docs/todo/backlog/task-998-git-mv-test.md << 'EOF'
# task-998: 测试 git mv 命令检测

**创建时间**: 2026-02-01
**优先级**: P1
**模块**: testing

## 任务描述
测试使用 git mv 命令移动任务卡片后，Hook 是否能正确检测并更新文档。
EOF

# 2. 使用 git mv 命令移动任务
git mv docs/todo/backlog/task-998-git-mv-test.md docs/done/2026-02/

# 3. 检查文档是否自动更新
cat docs/session.md | grep "task-998"
cat docs/todo/current-sprint.md | grep "task-998"
cat docs/done/archive-index.md | grep "task-998"
```

**预期结果**:
- ✅ `session.md` 的"上一个任务"表格中包含 task-998
- ✅ `current-sprint.md` 中 task-998 状态为 ✅ 完成
- ✅ `archive-index.md` 中包含 task-998 的归档记录

### 场景 3: 使用引号包裹路径的 mv 命令

**步骤**:
```bash
# 1. 创建测试任务
cat > docs/todo/backlog/task-997-quoted-path.md << 'EOF'
# task-997: 测试引号包裹路径

**创建时间**: 2026-02-01
**优先级**: P1
**模块**: testing

## 任务描述
测试使用引号包裹路径的 mv 命令。
EOF

# 2. 使用引号包裹路径移动任务
mv "docs/todo/backlog/task-997-quoted-path.md" "docs/done/2026-02/"

# 3. 检查文档是否自动更新
cat docs/session.md | grep "task-997"
```

**预期结果**:
- ✅ Hook 能正确解析带引号的路径
- ✅ 文档自动更新

### 场景 4: 新 Session 启动时检测未记录的任务

**步骤**:
```bash
# 1. 手动移动任务（模拟 Hook 失效的情况）
cat > docs/todo/backlog/task-996-manual-move.md << 'EOF'
# task-996: 手动移动测试

**创建时间**: 2026-02-01
**优先级**: P1
**模块**: testing
EOF

# 2. 使用 Bash 直接移动，不触发 Hook（如果 Hook 没有正常工作）
mv docs/todo/backlog/task-996-manual-move.md docs/done/2026-02/

# 3. 启动新的 Claude Code session
# SessionStart Hook 应该检测到未记录的任务

# 4. 检查输出
# 应该看到类似以下的提示：
# "🔍 检测到未记录的完成任务: task-996"
# "正在自动更新文档..."
```

**预期结果**:
- ✅ SessionStart Hook 检测到 task-996 未记录
- ✅ 自动更新 session.md、current-sprint.md、archive-index.md

## 故障排查

### 问题: mv 命令执行后文档未更新

**检查步骤**:

1. **验证 PostToolWrite Hook 是否加载**
```bash
cat ~/.claude/plugins/claude-task-pilot/hooks/post-tool-write.md | head -5
```

应该看到:
```markdown
---
when: PostToolWrite
ifTool: Write|Edit|mcp__plugin_serena_serena__replace_content|Bash
```

确认 `Bash` 在 `ifTool` 列表中。

2. **检查命令格式**
```bash
# 确保使用正确的命令格式
mv docs/todo/backlog/task-XXX.md docs/done/YYYY-MM/
```

不支持的模式:
```bash
# ❌ 不支持：通配符移动（难以检测具体任务）
mv docs/todo/backlog/*.md docs/done/2026-02/

# ❌ 不支持：使用变量
mv $source $target
```

3. **检查文件路径**
```bash
# 确保路径包含 docs/ 前缀
mv docs/todo/backlog/task-XXX.md docs/done/2026-02/

# 或者在项目根目录执行
mv docs/todo/backlog/task-XXX.md docs/done/2026-02/
```

4. **查看 Claude Code 日志**
```bash
# macOS
~/Library/Application\ Support/Claude/claude-code.log

# Linux
~/.config/Claude/claude-code.log

# Windows
%APPDATA%\Claude\claude-code.log
```

搜索 "PostToolWrite" 和 "TASK_COMPLETED" 关键词。

### 问题: SessionStart Hook 未检测到未记录的任务

**检查步骤**:

1. **验证任务确实未记录**
```bash
# 检查 archive-index.md
cat docs/done/archive-index.md | grep task-996

# 应该没有输出（表示未记录）
```

2. **验证任务文件存在**
```bash
# 检查归档文件
ls -la docs/done/2026-02/task-996-*.md
```

3. **手动触发检测**
```bash
# 重启 Claude Code session
# SessionStart Hook 应该自动执行检测
```

## 支持的命令格式

### ✅ 支持的格式

```bash
# 格式 1: 标准格式
mv docs/todo/backlog/task-001.md docs/done/2026-02/

# 格式 2: 引号包裹
mv "docs/todo/backlog/task-001.md" "docs/done/2026-02/"

# 格式 3: git mv
git mv docs/todo/backlog/task-001.md docs/done/2026-02/

# 格式 4: 相对路径（在项目根目录）
mv docs/todo/backlog/task-001.md docs/done/2026-02/
```

### ❌ 不支持的格式

```bash
# 不支持 1: 通配符
mv docs/todo/backlog/*.md docs/done/2026-02/

# 不支持 2: 变量替换
mv $SOURCE $TARGET

# 不支持 3: 复杂的管道
cat task-001.md | ... > docs/done/2026-02/task-001.md

# 不支持 4: 使用 cp 复制（不是移动）
cp docs/todo/backlog/task-001.md docs/done/2026-02/
```

## 修复建议

如果检测失败，可以手动更新文档：

```bash
# 1. 手动更新 session.md
# 添加到"## ⏳ 上一个任务"表格

# 2. 手动更新 current-sprint.md
# 将任务状态改为 ✅ 完成

# 3. 手动更新 archive-index.md
# 添加任务归档记录
```

或者使用 Write 工具移动文件（会触发 Write Hook）：
```bash
# 使用 Write 工具读取任务文件
# 然后在目标位置创建新文件
# 最后删除源文件
```

## 总结

修复后，用户可以使用以下任何方式完成任务，都会自动更新相关文档：

1. ✅ **Bash mv 命令** (新增)
2. ✅ **Bash git mv 命令** (新增)
3. ✅ **Write 工具移动** (原有)
4. ✅ **Edit 工具修改状态** (原有)
5. ✅ **SessionStart 自动检测** (新增)

确保所有场景都能正确更新文档，提供完整的任务管理体验。
