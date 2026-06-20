# 文档评审系统实现计划

> **致代理工作者：** 必需：使用 superpowers:subagent-driven-development（若子代理可用）或 superpowers:executing-plans 来实现本计划。

**目标：** 为 brainstorming 和 writing-plans 技能添加 spec 与 plan 文档的评审循环。

**架构：** 在每个技能目录中创建评审者提示模板。修改技能文件，在文档创建后加入评审循环。使用 Task 工具配合 general-purpose 子代理来调度评审者。

**技术栈：** Markdown 技能文件、通过 Task 工具调度子代理

**规格：** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## Chunk 1: Spec 文档评审者

本块为 brainstorming 技能添加 spec 文档评审者。

### Task 1: 创建 Spec 文档评审者提示模板

**文件：**
- 创建：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **Step 1:** 创建评审者提示模板文件

```markdown
# Spec Document Reviewer Prompt Template

在调度 spec 文档评审者子代理时使用此模板。

**目的：** 验证 spec 是否完整、一致，并且已准备好进入实现规划阶段。

**调度时机：** Spec 文档被写入 docs/superpowers/specs/ 之后

```
Task tool (general-purpose):
  description: "Review spec document"
  prompt: |
    You are a spec document reviewer. Verify this spec is complete and ready for planning.

    **Spec to review:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, "TBD", incomplete sections |
    | Coverage | Missing error handling, edge cases, integration points |
    | Consistency | Internal contradictions, conflicting requirements |
    | Clarity | Ambiguous requirements |
    | YAGNI | Unrequested features, over-engineering |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Sections saying "to be defined later" or "will spec when X is done"
    - Sections noticeably less detailed than others

    ## Output Format

    ## Spec Review

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Section X]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**评审者返回：** Status、Issues（若有）、Recommendations
```

- [ ] **Step 2:** 验证文件已正确创建

运行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **Step 3:** 提交

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### Task 2: 在 brainstorming 技能中添加评审循环

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **Step 1:** 读取当前 brainstorming 技能

运行：`cat skills/brainstorming/SKILL.md`

- [ ] **Step 2:** 在"After the Design"之后添加评审循环部分

找到"After the Design"部分，在文档之后、实现之前新增一个"Spec Review Loop"部分：

```markdown
**Spec Review Loop:**
After writing the spec document:
1. Dispatch spec-document-reviewer subagent (see spec-document-reviewer-prompt.md)
2. If ❌ Issues Found:
   - Fix the issues in the spec document
   - Re-dispatch reviewer
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to implementation setup

**Review loop guidance:**
- Same agent that wrote the spec fixes it (preserves context)
- If loop exceeds 5 iterations, surface to human for guidance
- Reviewers are advisory - explain disagreements if you believe feedback is incorrect
```

- [ ] **Step 3:** 验证改动

运行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
预期：显示新的评审循环部分

- [ ] **Step 4:** 提交

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## Chunk 2: Plan 文档评审者

本块为 writing-plans 技能添加 plan 文档评审者。

### Task 3: 创建 Plan 文档评审者提示模板

**文件：**
- 创建：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **Step 1:** 创建评审者提示模板文件

```markdown
# Plan Document Reviewer Prompt Template

在调度 plan 文档评审者子代理时使用此模板。

**目的：** 验证 plan 的某个 chunk 是否完整、与 spec 一致，并且任务拆分合理。

**调度时机：** 每个 plan chunk 写完之后

```
Task tool (general-purpose):
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TODOs, placeholders, incomplete tasks, missing steps |
    | Spec Alignment | Chunk covers relevant spec requirements, no scope creep |
    | Task Decomposition | Tasks atomic, clear boundaries, steps actionable |
    | Task Syntax | Checkbox syntax (`- [ ]`) on tasks and steps |
    | Chunk Size | Each chunk under 1000 lines |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing verification steps or expected outputs

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** ✅ Approved | ❌ Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**评审者返回：** Status、Issues（若有）、Recommendations
```

- [ ] **Step 2:** 验证文件已创建

运行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
预期：显示标题和目的部分

- [ ] **Step 3:** 提交

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### Task 4: 在 writing-plans 技能中添加评审循环

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 读取当前技能文件

运行：`cat skills/writing-plans/SKILL.md`

- [ ] **Step 2:** 添加按 chunk 评审的部分

在"Execution Handoff"部分之前添加：

```markdown
## Plan Review Loop

After completing each chunk of the plan:

1. Dispatch plan-document-reviewer subagent for the current chunk
   - Provide: chunk content, path to spec document
2. If ❌ Issues Found:
   - Fix the issues in the chunk
   - Re-dispatch reviewer for that chunk
   - Repeat until ✅ Approved
3. If ✅ Approved: proceed to next chunk (or execution handoff if last chunk)

**Chunk boundaries:** Use `## Chunk N: <name>` headings to delimit chunks. Each chunk should be ≤1000 lines and logically self-contained.
```

- [ ] **Step 3:** 更新任务语法示例以使用复选框

将 Task Structure 部分改为展示复选框语法：

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **Step 4:** 验证评审循环部分已添加

运行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
预期：显示新的评审循环部分

- [ ] **Step 5:** 验证任务语法示例已更新

运行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
预期：显示复选框语法 `### Task N:`

- [ ] **Step 6:** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## Chunk 3: 更新 Plan 文档头部

本块更新 plan 文档头部模板，使其引用新的复选框语法要求。

### Task 5: 在 writing-plans 技能中更新 Plan 头部模板

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 读取当前 plan 头部模板

运行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **Step 2:** 更新头部模板以引用复选框语法

plan 头部应注明任务和步骤使用复选框语法。更新头部注释：

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Tasks and steps use checkbox (`- [ ]`) syntax for tracking.
```

- [ ] **Step 3:** 验证改动

运行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
预期：显示已更新、提及复选框语法的头部

- [ ] **Step 4:** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```
