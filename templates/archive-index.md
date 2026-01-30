# 已完成任务归档索引

**用途**: 快速查找历史任务和文档的完成情况

---

## {{CURRENT_YEAR}}年{{CURRENT_MONTH}}月

### Week {{CURRENT_WEEK}}（{{CURRENT_WEEK_DATES}}）

| 日期 | 完成任务 | 关键成果 | 归档位置 |
|------|----------|----------|----------|
{{CURRENT_WEEK_COMPLETION}}

---

## 快速查找

### {{CATEGORY_1}}
{{CATEGORY_1_ITEMS}}

### {{CATEGORY_2}}
{{CATEGORY_2_ITEMS}}

---

## 归档说明

### 文件组织
- **{{CURRENT_YEAR}}-{{CURRENT_MONTH}}/**: {{CURRENT_YEAR}}年{{CURRENT_MONTH}}月完成的任务
- **{{NEXT_YEAR}}-{{NEXT_MONTH}}/**: {{NEXT_YEAR}}年{{NEXT_MONTH}}月完成的任务（待创建）

### 归档规则
1. 每月创建一个文件夹 `done/YYYY-MM/`
2. 任务完成后创建完成报告 `task-XXX-{description}.md`
3. 历史文档保留在原位置，此索引文件提供快速查找
4. 大型文档保留参考，不移动

### 命名规范
- 任务报告: `task-{ID}-{简短描述}.md`
- 周总结: `week-{N}-summary.md`
- 月总结: `YYYY-MM-summary.md`

---

## 统计数据

### {{CURRENT_YEAR}}年{{CURRENT_MONTH}}月
- **完成任务**: {{COMPLETED_TASK_COUNT}}个
- **主要成果**:
{{KEY_ACHIEVEMENTS}}

### 关键里程碑
{{KEY_MILESTONES}}

---

**最后更新**: {{LAST_UPDATE_DATE}}
**下次更新**: {{NEXT_UPDATE_DATE}}
