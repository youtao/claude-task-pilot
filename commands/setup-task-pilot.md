---
description: Initialize Claude Task Pilot for mid-development projects with smart file detection
argument-hint: Optional mode (auto/interactive)
allowed-tools: ["Read", "Write", "Bash", "AskUserQuestion", "Glob", "Grep", "ListMcpResourcesTool"]
---

# Claude Task Pilot 初始化

为已开发的项目初始化 Claude Task Pilot，智能检测并创建缺失的文档文件。

**当前模式**: $ARGUMENTS

## 初始化步骤

1. **检测插件根目录**:
   - 使用 `${CLAUDE_PLUGIN_ROOT}` 环境变量
   - 验证模板文件路径

2. **检测项目结构**:
   ```bash
   # 检查 docs 目录
   ls -la docs/ 2>/dev/null || echo "docs 目录不存在"
   ```

3. **智能文件处理**:

   对于每个模板文件，执行：
   - **文件不存在**: 直接创建
   - **文件存在且 < 5 行**: 询问用户是否覆盖（可能初始化失败）
   - **文件存在且 ≥ 5 行**: 跳过（保留用户内容）

4. **创建目录结构**:
   ```bash
   mkdir -p docs/todo/backlog
   mkdir -p docs/done/$(date +%Y-%m)
   ```

5. **处理模板文件**:

   使用 `Glob` 查找模板：
   ```
   ${CLAUDE_PLUGIN_ROOT}/templates/*.md
   ```

   对每个模板：
   - 读取模板内容
   - 检查目标文件是否存在
   - 根据行数决定：创建/询问/跳过

6. **创建 session.md**:
   - 检查 `docs/session.md`
   - 如果需要创建，从模板初始化

7. **创建 roadmap.md**:
   - 检查 `docs/todo/roadmap.md`
   - 如果需要创建，从模板初始化

8. **创建 current-sprint.md**:
   - 检查 `docs/todo/current-sprint.md`
   - 如果需要创建，从模板初始化

9. **创建 archive-index.md**:
   - 检查 `docs/done/archive-index.md`
   - 如果需要创建，从模板初始化

## 特殊场景处理

**交互模式** (interactive):
- 对每个可能覆盖的文件使用 `AskUserQuestion` 确认
- 询问：文件 X 已存在但内容较少，是否重新初始化？

**自动模式** (auto):
- 自动判断：行数 < 5 视为失败并重建
- 行数 ≥ 5 视为有效并保留

## 错误处理

- 模板文件缺失：提示用户重新安装插件
- 目录创建失败：检查权限
- 文件写入失败：提示错误并跳过

## 完成提示

初始化完成后，输出：
```
✅ Claude Task Pilot 初始化完成

创建的文件：
- docs/session.md
- docs/todo/roadmap.md
- docs/todo/current-sprint.md
- docs/done/archive-index.md

下一步：
1. 在 docs/todo/roadmap.md 中规划长期目标
2. 在 docs/todo/current-sprint.md 中定义当前冲刺
3. 创建任务卡片到 docs/todo/backlog/
```

---
**注意**: 此命令会保留已有的丰富内容，只初始化缺失或明显失败的文件。
