---
name: writing-plans
description: 当你有多步任务的规格或需求、在动代码之前使用
---

# 编写计划

## 概览

编写全面的实现计划，假设工程师对我们的代码库零上下文，且品味堪忧。记录他们需要知道的一切：每个任务要改哪些文件、代码、测试、他们可能需要查的文档、如何测试。把整个计划作为细粒度任务给他们。DRY。YAGNI。TDD。频繁提交。

假设他们是熟练的开发者，但几乎不了解我们的工具集或问题领域。假设他们不太懂好的测试设计。

**开始时宣告：** "我在用 writing-plans 技能来创建实现计划。"

**上下文：** 如果在隔离的 worktree 中工作，它应该在执行时通过 `superpowers:using-git-worktrees` 技能创建。

**计划保存到：** `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`
- （用户对计划位置的偏好优先于此默认值）

## 范围检查

如果规格覆盖多个独立子系统，它本应在头脑风暴期间被拆成子项目规格。如果没有，建议把它拆成单独的计划——每个子系统一个。每个计划应能独立产出可工作、可测试的软件。

## 文件结构

在定义任务之前，规划出哪些文件会被创建或修改，以及每个负责什么。这是锁定拆解决策的地方。

- 设计有清晰边界和定义良好接口的单元。每个文件应有一个清晰的职责。
- 你对能一次装入上下文的代码推理得更好，当文件聚焦时你的编辑也更可靠。偏好更小、聚焦的文件，而非做太多事的大文件。
- 一起改动的文件应放在一起。按职责拆分，而非按技术层。
- 在既有代码库中，遵循既有模式。如果代码库用大文件，不要单方面重构——但如果你正在修改的文件已经难以驾驭，在计划中包含一次拆分是合理的。

这个结构指导任务拆解。每个任务应产出独立成理、能独立理解的改动。

## 任务大小判定

一个任务是承载其自身测试周期、且值得一个全新评审者关卡的最小单元。画任务边界时：把 setup、配置、脚手架和文档步骤折叠进需要它们的那个任务的交付物里；只在评审者能有意义地否决一个任务而批准其邻居时才拆分。每个任务以一个可独立测试的交付物结束。

## 细粒度任务粒度

**每一步是一个动作（2-5 分钟）：**
- "写失败测试" —— 一步
- "运行它确认失败" —— 一步
- "实现让测试通过的最小代码" —— 一步
- "运行测试确认通过" —— 一步
- "提交" —— 一步

## 计划文档头部

**每个计划必须以此头部开始：**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits,
naming and copy rules, platform requirements — one line each, with exact
values copied verbatim from the spec. Every task's requirements implicitly
include this section.]

---
```

## 任务结构

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter
  and return types. A task's implementer sees only their own task; this
  block is how they learn the names and types neighboring tasks use.]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## 不要占位符

每一步必须包含工程师需要的实际内容。这些是**计划失败**——绝不要写它们：
- "TBD"、"TODO"、"以后实现"、"填充细节"
- "加恰当的错误处理" / "加校验" / "处理边界情况"
- "为以上写测试"（没有实际测试代码）
- "与任务 N 类似"（重复代码——工程师可能乱序读任务）
- 描述做什么却不展示怎么做的步骤（代码步骤需要代码块）
- 引用未在任何任务中定义的类型、函数或方法

## 记住
- 总是确切的文件路径
- 每一步都有完整代码——如果某步改代码，展示代码
- 确切命令带预期输出
- DRY、YAGNI、TDD、频繁提交

## 自审

写完完整计划后，用新的眼光看规格，对照它检查计划。这是你自己跑的清单——不是子代理分派。

**1. 规格覆盖：** 略读规格的每个章节/需求。你能指向实现它的任务吗？列出任何缺口。

**2. 占位符扫描：** 搜索你计划里的红旗——上面"不要占位符"章节的任何模式。修掉它们。

**3. 类型一致性：** 你在后续任务中用的类型、方法签名和属性名是否匹配你在更早任务中定义的？一个在任务 3 里叫 `clearLayers()` 而在任务 7 里叫 `clearFullLayers()` 的函数是 bug。

如果你发现问题，就地修。无需复查——修完继续。如果你发现一个规格需求没有任务，加上任务。

## 执行交接

保存计划后，提供执行选择：

**"计划完成并保存到 `docs/superpowers/plans/<filename>.md`。两种执行选项：**

**1. 子代理驱动（推荐）** - 我每个任务分派一个全新子代理，任务之间评审，快迭代

**2. 内联执行** - 在本会话中用 executing-plans 执行任务，带检查点的批量执行

**选哪种？"**

**如果选子代理驱动：**
- **必需子技能：** 用 superpowers:subagent-driven-development
- 每个任务全新子代理 + 两阶段评审

**如果选内联执行：**
- **必需子技能：** 用 superpowers:executing-plans
- 带检查点的批量执行以供评审
