---
description: 将设计文档转换为任务卡片
argument-hint: 设计文档路径（可选，默认使用最新）
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

# Convert Design to Tasks - 设计文档转换命令

## 功能描述

将设计文档自动转换为可执行的任务卡片，支持手动触发和调试。

---

## 参数说明

**$ARGUMENTS**: 设计文档路径（可选）

- **未提供**: 自动查找最新的设计文档
- **提供路径**: 使用指定的设计文档
- **相对路径**: 相对于项目根目录
- **绝对路径**: 使用完整路径

---

## 执行流程

### 步骤 1: 确定设计文档

```javascript
// 如果未提供参数
if (!ARGUMENTS || ARGUMENTS.trim() === '') {
  // 使用 Glob 查找最新设计文档
  const designDocs = await glob('docs/plans/*-design.md')

  if (designDocs.length === 0) {
    return {
      success: false,
      error: '未找到设计文档',
      message: '期望路径: docs/plans/YYYY-MM-DD-<topic>-design.md'
    }
  }

  // 按修改时间排序，选择最新的
  designDocs.sort((a, b) => {
    const statA = await fs.stat(a)
    const statB = await fs.stat(b)
    return statB.mtime - statA.mtime
  })

  designDoc = designDocs[0]
  console.log(`使用最新设计文档: ${designDoc}`)
} else {
  // 使用提供的路径
  designDoc = ARGUMENTS

  // 支持相对路径
  if (!designDoc.startsWith('/') && !designDoc.match(/^[A-Z]:/i)) {
    designDoc = path.join(process.cwd(), designDoc)
  }
}
```

### 步骤 2: 验证文件

```javascript
// 检查文件是否存在
try {
  await fs.access(designDoc)
} catch {
  return {
    success: false,
    error: '设计文档不存在',
    path: designDoc,
    message: '请检查文件路径是否正确'
  }
}

// 检查文件格式
if (!designDoc.endsWith('.md')) {
  return {
    success: false,
    error: '文件格式不支持',
    message: '设计文档必须是 Markdown 格式（.md）'
  }
}

// 检查是否是设计文档
if (!designDoc.match(/\/\d{4}-\d{2}-\d{2}-.+-design\.md$/)) {
  console.log('⚠️ 警告: 文件名不符合设计文档命名规范')
  console.log('期望格式: YYYY-MM-DD-<topic>-design.md')
  console.log('是否继续？ (Y/n)')

  // 询问用户
  const confirm = await askUser('是否继续？')
  if (!confirm) {
    return { success: false, error: '用户取消操作' }
  }
}
```

### 步骤 3: 读取并分析设计文档

```javascript
// 读取文件内容
const content = await readFile(designDoc, 'utf-8')

// 检查内容是否为空
if (!content || content.trim().length < 50) {
  return {
    success: false,
    error: '设计文档内容为空',
    message: '请完善设计文档内容，至少包含功能描述和技术方案'
  }
}

// 显示文档信息
const basename = path.basename(designDoc)
console.log(`\n📋 分析设计文档: ${basename}`)
console.log(`路径: ${designDoc}`)
console.log(`大小: ${content.length} 字符\n`)
```

### 步骤 4: 确认转换

```javascript
console.log('准备将设计文档转换为任务卡片...')
console.log('任务将创建到: docs/todo/backlog/\n')

const confirm = await askUser('是否继续？ (Y/n)', { default: true })

if (!confirm) {
  return { success: false, error: '用户取消操作' }
}
```

### 步骤 5: 调用 Design-to-Tasks Agent

```javascript
// 调用 agent 执行转换
const result = await runAgent('design-to-tasks', {
  designDoc: designDoc,
  mode: 'manual',  // 手动模式，输出更详细
  verbose: true
})

return result
```

### 步骤 6: 显示结果

```javascript
if (result.success) {
  console.log('\n' + '='.repeat(60))
  console.log('✅ 转换完成！')
  console.log('='.repeat(60))

  console.log(`\n生成任务: ${result.taskCount} 个`)
  console.log(`任务ID: ${result.taskIds.join(', ')}`)

  console.log('\n下一步:')
  console.log(`1. 查看任务列表: cat docs/todo/current-sprint.md`)
  console.log(`2. 获取推荐: 说"推荐下一个任务"`)
  console.log(`3. 开始任务: cat docs/todo/backlog/${result.firstTask}.md`)
} else {
  console.log('\n❌ 转换失败')
  console.log(`错误: ${result.error}`)
  console.log(`建议: ${result.suggestion || '请检查设计文档格式'}`)
}
```

---

## 使用示例

### 示例 1: 使用最新设计文档

```bash
/convert-design-to-tasks
```

**输出**:
```markdown
查找最新设计文档...

找到: docs/plans/2026-02-01-user-authentication-design.md
修改时间: 2026-02-01 10:30:00

准备将设计文档转换为任务卡片...
任务将创建到: docs/todo/backlog/

是否继续？ (Y/n) y

[调用 agent 转换...]

✅ 转换完成！
生成任务: 5 个
任务ID: task-001, task-002, task-003, task-004, task-005

下一步:
1. 查看任务列表: cat docs/todo/current-sprint.md
2. 获取推荐: 说"推荐下一个任务"
3. 开始任务: cat docs/todo/backlog/task-001-design-user-data-model.md
```

### 示例 2: 指定设计文档

```bash
/convert-design-to-tasks docs/plans/2026-02-01-user-authentication-design.md
```

### 示例 3: 使用相对路径

```bash
/convert-design-to-tasks plans/2026-02-01-user-authentication-design.md
```

### 示例 4: 使用绝对路径

```bash
/convert-design-to-tasks /home/user/project/docs/plans/2026-02-01-user-authentication-design.md
```

---

## 错误处理

### E1. 未找到设计文档

```markdown
❌ 未找到设计文档

期望路径: docs/plans/YYYY-MM-DD-<topic>-design.md

检查操作:
1. 确认设计文档已创建
2. 检查路径是否正确
3. 查看现有文档: ls -la docs/plans/
```

### E2. 设计文档不存在

```markdown
❌ 设计文档不存在

路径: docs/plans/2026-02-01-unknown-design.md

可能的原因:
1. 文件路径拼写错误
2. 文件已被移动或删除
3. 路径格式不正确

建议:
1. 使用完整路径
2. 或不提供参数，自动查找最新文档
```

### E3. 文件格式不支持

```markdown
❌ 文件格式不支持

设计文档必须是 Markdown 格式（.md）

当前文件: design.txt

建议: 将文件重命名为 .md 后缀
```

### E4. 设计文档内容为空

```markdown
❌ 设计文档内容为空或过于简单

当前内容: {{content_length}} 字符

期望的设计文档应包含:
- 功能描述（概述、背景）
- 技术方案（架构、组件）
- 实现细节（API、数据模型等）

建议:
1. 完善设计文档内容
2. 使用 /brainstorm 重新生成
```

### E5. backlog 目录不存在

```markdown
⚠️ 任务管理目录未初始化

正在自动初始化...

✅ 已创建: docs/todo/backlog/
✅ 已创建: docs/done/2026-02/

继续生成任务...
```

### E6. 任务ID冲突

```markdown
⚠️ 任务ID冲突: task-005 已存在

冲突文件: docs/todo/backlog/task-005-existing-task.md

解决方案:
1. 跳过此ID，使用下一个可用ID
2. 删除现有文件并重新创建
3. 取消操作

请选择 (1/2/3):
```

---

## 调试模式

### 显示详细日志

```bash
# 设置环境变量启用调试
DEBUG=claude-task-pilot:* /convert-design-to-tasks
```

### 验证设计文档

```bash
# 只验证设计文档格式，不生成任务
/convert-design-to-tasks --validate
```

### 显示任务预览

```bash
# 生成任务预览，不创建文件
/convert-design-to-tasks --dry-run
```

---

## 配置选项

命令行为受 `.claude/claude-task-pilot.local.md` 配置影响：

```yaml
---
# 设计文档路径
design_docs_path: "docs/plans"

# 默认优先级
default_priority: "P1"

# 默认里程碑
default_milestone: "Current Sprint"

# 任务复杂度阈值（小时）
task_complexity_threshold: 4
---
```

---

## 与 Hook 的区别

| 特性 | 手动命令 | Hook 自动触发 |
|------|---------|--------------|
| 触发方式 | 用户主动调用 | 文件创建时自动 |
| 参数支持 | 支持指定文档 | 使用创建的文档 |
| 询问确认 | 每次都询问 | 可配置是否询问 |
| 调试友好 | 是 | 否 |
| 适用场景 | 测试、调试、重试 | 日常开发流程 |

---

## 最佳实践

1. **首次使用**: 先使用 `--dry-run` 预览任务
2. **调整结果**: 生成后可手动编辑任务卡片
3. **版本控制**: 提交任务卡片到 Git
4. **定期清理**: 及时归档已完成的任务

---

## 相关命令

- `/setup-task-pilot` - 初始化项目结构
- `/suggest-next-task` - 智能推荐下一个任务
- `/convert-design-to-tasks` - 本命令

---

**版本**: 1.0.0
**最后更新**: 2026-02-01
