# 模板变量命名规范

**版本**: 1.0.0
**最后更新**: 2026-01-30

---

## 📋 命名原则

1. **一致性**: 所有模板使用统一的变量名
2. **描述性**: 变量名清晰表达含义
3. **层次化**: 使用 `_` 分隔不同层次
4. **大小写**: 全部大写，便于识别

---

## 🎯 通用变量

### 时间相关

| 变量名 | 格式 | 示例 | 说明 |
|--------|------|------|------|
| `{{CURRENT_DATE}}` | YYYY-MM-DD | 2026-01-30 | 当前日期 |
| `{{CURRENT_TIME}}` | HH:MM:SS | 14:30:45 | 当前时间 |
| `{{CURRENT_DATETIME}}` | YYYY-MM-DD HH:MM | 2026-01-30 14:30 | 日期+时间 |
| `{{TIMESTAMP}}` | ISO 8601 | 2026-01-30T14:30:45Z | 完整时间戳 |
| `{{LAST_UPDATE_DATE}}` | YYYY-MM-DD | 2026-01-30 | 最后更新日期 |
| `{{LAST_UPDATE_DATETIME}}` | YYYY-MM-DD HH:MM | 2026-01-30 14:30 | 最后更新时间 |

### 项目相关

| 变量名 | 类型 | 示例 | 说明 |
|--------|------|------|------|
| `{{PROJECT_NAME}}` | string | Claude Task Pilot | 项目名称 |
| `{{PROJECT_POSITIONING}}` | string | AI驱动的任务管理... | 项目定位 |
| `{{MVP_GOAL}}` | string | 实现自动化任务追踪 | MVP目标 |

---

## 📝 任务相关变量

### 任务基本信息

| 变量名 | 类型 | 示例 | 说明 |
|--------|------|------|------|
| `{{TASK_ID}}` | string | task-003 | 任务ID |
| `{{TASK_NAME}}` | string | 补充番茄品种数据 | 任务名称 |
| `{{TASK_DESCRIPTION}}` | string | 补充50个常见番茄... | 任务描述 |
| `{{TASK_OVERVIEW}}` | string | 本任务的目标是... | 任务概述 |

### 任务状态

| 变量名 | 可能值 | 说明 |
|--------|--------|------|
| `{{TASK_STATUS}}` | ⏳ 待开始 / 🔄 进行中 / ✅ 完成 / ❌ 取消 | 任务状态 |
| `{{TASK_PRIORITY}}` | P0 / P1 / P2 | 任务优先级 |
| `{{TASK_MODULE}}` | backend / frontend / ai / test | 负责模块 |

### 任务时间

| 变量名 | 格式 | 说明 |
|--------|------|------|
| `{{CREATION_DATE}}` | YYYY-MM-DD | 创建日期 |
| `{{CREATION_DATETIME}}` | YYYY-MM-DD HH:MM | 创建时间 |
| `{{COMPLETION_DATE}}` | YYYY-MM-DD | 完成日期 |
| `{{ESTIMATED_TIME}}` | 2-3小时 / 1天 | 预计耗时 |
| `{{ACTUAL_TIME}}` | 3小时 / 1.5天 | 实际耗时 |

### 任务关系

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{TASK_DEPENDENCIES}}` | array | 依赖任务列表 |
| `{{TASK_MILESTONE}}` | string | 所属里程碑 |
| `{{RELATED_TASKS}}` | array | 相关任务 |

---

## 👤 人员相关

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{TASK_OWNER}}` | string | 任务负责人 |
| `{{SPRINT_OWNER}}` | string | Sprint负责人 |
| `{{AUTHOR}}` | string | 作者 |

---

## 📊 进度相关

### Sprint 进度

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{SPRINT_NUMBER}}` | integer | Sprint编号 |
| `{{SPRINT_TITLE}}` | string | Sprint标题 |
| `{{SPRINT_START_DATE}}` | YYYY-MM-DD | Sprint开始日期 |
| `{{SPRINT_END_DATE}}` | YYYY-MM-DD | Sprint结束日期 |
| `{{SPRINT_DURATION}}` | string | 2周 / 10天 | Sprint持续时间 |
| `{{SPRINT_GOAL}}` | string | Sprint目标 |
| `{{SPRINT_PROGRESS}}` | integer | 0-100 | Sprint进度百分比 |

### 周相关

| 变量名 | 格式 | 说明 |
|--------|------|------|
| `{{CURRENT_WEEK}}` | integer | 1-52 | 当前周数 |
| `{{CURRENT_WEEK_DATES}}` | YYYY-MM-DD ~ YYYY-MM-DD | 本周日期范围 |
| `{{NEXT_WEEK}}` | integer | 下周周数 |
| `{{NEXT_WEEK_DATES}}` | YYYY-MM-DD ~ YYYY-MM-DD | 下周日期范围 |

---

## 📁 路径相关

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{DOCS_ROOT}}` | string | 文档根目录 |
| `{{ARCHIVE_PATH}}` | string | 归档文件路径 |
| `{{PREV_TASK_LOCATION}}` | string | 上一个任务归档位置 |

---

## ✅ 完成报告相关

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{MAIN_ACHIEVEMENTS}}` | list | 主要成果列表 |
| `{{KEY_CODE_CHANGES}}` | list | 关键代码变更 |
| `{{FILE_LIST}}` | list | 文件清单 |
| `{{KEY_DECISIONS}}` | list | 关键决策 |
| `{{TECH_SELECTIONS}}` | list | 技术选型 |
| `{{PROBLEMS_AND_SOLUTIONS}}` | list | 问题和解决方案 |
| `{{TEST_SCENARIOS}}` | list | 测试场景 |
| `{{TEST_RESULTS}}` | list | 测试结果 |
| `{{KNOWN_ISSUES}}` | list | 已知问题 |
| `{{NEXT_TASKS}}` | list | 下一步任务 |
| `{{IMPROVEMENT_SUGGESTIONS}}` | list | 改进建议 |
| `{{SUCCESS_LESSONS}}` | list | 成功经验 |
| `{{LESSONS_LEARNED}}` | list | 踩坑记录 |
| `{{BEST_PRACTICES}}` | list | 最佳实践 |

### 数据统计

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{LINES_OF_CODE}}` | integer | 代码行数 |
| `{{FILE_COUNT}}` | integer | 文件数量 |
| `{{COMMIT_COUNT}}` | integer | 提交次数 |
| `{{TEST_COVERAGE}}` | string | 测试覆盖率（如 85%） |
| `{{COMPLETED_TASK_COUNT}}` | integer | 完成任务数 |

---

## 🔗 路线图相关

### 季度/阶段

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{TIMELINE}}` | string | 时间线（如 Q1 2026） |
| `{{QUARTER_1}}` | string | 第一季度标识 |
| `{{Q1_DATES}}` | string | Q1日期范围 |
| `{{Q1_OBJECTIVES}}` | list | Q1目标 |
| `{{Q2_DATES}}` | string | Q2日期范围 |
| `{{Q2_OBJECTIVES}}` | list | Q2目标 |

### 里程碑

| 变量名 | 类型 | 说明 |
|--------|------|------|
| `{{PHASE_N_NAME}}` | string | 阶段名称 |
| `{{PHASE_N_DATES}}` | string | 阶段日期范围 |
| `{{PHASE_N_OBJECTIVES}}` | list | 阶段目标 |
| `{{PHASE_N_COMPLETED}}` | list | 阶段完成内容 |
| `{{PHASE_N_OUTCOMES}}` | list | 阶段成果 |
| `{{PHASE_N_TASKS}}` | list | 阶段任务列表 |
| `{{NEXT_MILESTONE}}` | string | 下一个里程碑 |

---

## 📦 列表变量规范

### 单行列表格式

```markdown
{{ITEM_1}}
{{ITEM_2}}
{{ITEM_3}}
```

### 表格列表格式

```markdown
| 列1 | 列2 | 列3 |
|-----|-----|-----|
{{TABLE_ROWS}}
```

其中 `{{TABLE_ROWS}}` 应为：

```markdown
| 值1 | 值2 | 值3 |
| 值4 | 值5 | 值6 |
```

---

## 🎨 特殊格式

### 复选框列表

```markdown
{{CHECKLIST_ITEMS}}
```

格式：

```markdown
- [ ] 待办事项1
- [x] 已完成事项
- [ ] 待办事项2
```

### 代码块

```markdown
{{{CODE_BLOCK}}}
```

使用三重大括号 `{{{` 避免被模板引擎解析。

---

## 📝 使用示例

### session.md 模板

```markdown
# 当前 Session 任务状态

**更新时间**: {{CURRENT_DATETIME}}
**Session ID**: session-{{CURRENT_DATE}}-{{RANDOM_ID}}

## 🎯 当前任务

| 任务ID | 描述 | 状态 | 负责模块 | 优先级 |
|--------|------|------|----------|--------|
| {{TASK_ID}} | {{TASK_NAME}} | {{TASK_STATUS}} | {{TASK_MODULE}} | {{TASK_PRIORITY}} |

## ⏳ 上一个任务

| 任务ID | 描述 | 完成时间 | 归档位置 |
|--------|------|----------|----------|
| {{PREV_TASK_ID}} | {{PREV_TASK_NAME}} | {{COMPLETION_DATE}} | {{PREV_TASK_LOCATION}} |
```

### task-complete.md 模板

```markdown
# {{TASK_ID}}: {{TASK_NAME}} - 完成报告

**完成时间**: {{COMPLETION_DATETIME}}
**负责人**: {{TASK_OWNER}}
**预计时间**: {{ESTIMATED_TIME}}
**实际时间**: {{ACTUAL_TIME}}

## ✅ 完成内容

{{MAIN_ACHIEVEMENTS}}

## 💡 技术决策

{{KEY_DECISIONS}}

## 📊 数据统计

- **代码行数**: {{LINES_OF_CODE}}
- **文件数量**: {{FILE_COUNT}}
- **提交次数**: {{COMMIT_COUNT}}
```

---

## ✅ 检查清单

添加新变量时，确保：

- [ ] 变量名全大写
- [ ] 使用 `_` 分隔单词
- [ ] 变量名具有描述性
- [ ] 在本文档中记录
- [ ] 在所有相关模板中更新
- [ ] 提供使用示例

---

**维护者**: youtao
**最后审查**: 2026-01-30
