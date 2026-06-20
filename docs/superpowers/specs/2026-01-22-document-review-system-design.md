# 文档评审系统设计

## 概述

为 superpowers 工作流新增两个评审阶段：

1. **规格文档评审** — 在 brainstorming 之后、writing-plans 之前
2. **计划文档评审** — 在 writing-plans 之后、实现之前

两者都遵循实现评审所用的迭代循环模式。

## 规格文档评审者

**目的：** 验证规格是否完整、一致、已为实现规划做好准备。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 检查点 |
|----------|------------------|
| 完整性 | TODO、占位符、"TBD"、不完整的章节 |
| 覆盖度 | 缺失的错误处理、边界情形、集成点 |
| 一致性 | 内部矛盾、相互冲突的需求 |
| 清晰度 | 含糊的需求 |
| YAGNI | 未被要求的功能、过度设计 |

**输出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues (if any):**
- [Section X]: [issue] - [why it matters]

**Recommendations (advisory):**
- [suggestions that don't block approval]
```

**评审循环：** 发现问题 -> brainstorming 代理修复 -> 重新评审 -> 重复直到通过。

**派发机制：** 使用 Task 工具，`subagent_type: general-purpose`。评审者提示模板提供完整提示。brainstorming 技能的控制器负责派发评审者。

## 计划文档评审者

**目的：** 验证计划是否完整、与规格匹配，并具备合理的任务拆分。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 检查点 |
|----------|------------------|
| 完整性 | TODO、占位符、不完整的任务 |
| 规格对齐 | 计划覆盖规格需求、无范围蔓延 |
| 任务拆分 | 任务原子化、边界清晰 |
| 任务语法 | 任务和步骤上的复选框语法 |
| 分块大小 | 每块不超过 1000 行 |

**分块定义：** 一个块是计划文档内任务的逻辑分组，以 `## Chunk N: <name>` 标题分隔。writing-plans 技能按逻辑阶段（例如 "Foundation"、"Core Features"、"Integration"）划分这些边界。每个块应足够自包含，能被独立评审。

**规格对齐验证：** 评审者会同时收到：
1. 计划文档（或当前块）
2. 规格文档的路径作为参考

评审者阅读两者并比较需求覆盖。

**输出格式：** 与规格评审者相同，但作用域限定为当前块。

**评审流程（逐块）：**
1. writing-plans 创建块 N
2. 控制器把块 N 内容和规格路径派发给 plan-document-reviewer
3. 评审者读块与规格，给出裁定
4. 若有问题：writing-plans 代理修复块 N，转到步骤 2
5. 若通过：进入块 N+1
6. 重复直到所有块通过

**派发机制：** 与规格评审者相同 — 使用 Task 工具，`subagent_type: general-purpose`。

## 更新后的工作流

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**规格评审循环：**
1. 规格完成
2. 派发评审者
3. 若有问题：修复 -> 转到 2
4. 若通过：继续

**计划评审循环：**
1. 块 N 完成
2. 为块 N 派发评审者
3. 若有问题：修复 -> 转到 2
4. 若通过：下一块或进入实现

## Markdown 任务语法

任务和步骤使用复选框语法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 错误处理

**评审循环终止：**
- 无硬性迭代上限 — 循环持续到评审者通过
- 若循环超过 5 次迭代，控制器应将其上报给人类寻求指导
- 人类可选择：继续迭代、在已知问题下通过、或中止

**分歧处理：**
- 评审者是顾问性质 — 他们标记问题但不阻塞
- 若代理认为评审者反馈不正确，应在修复时说明理由
- 若同一问题在 3 次迭代后仍存在分歧，上报给人类

**评审者输出畸形：**
- 控制器应校验评审者输出包含必需字段（Status、以及适用时的 Issues）
- 若畸形，附上关于预期格式的说明重新派发评审者
- 出现 2 次畸形响应后，上报给人类

## 待改文件

**新文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改文件：**
- `skills/brainstorming/SKILL.md` — 规格写完后加入评审循环
- `skills/writing-plans/SKILL.md` — 加入逐块评审循环，更新任务语法示例
