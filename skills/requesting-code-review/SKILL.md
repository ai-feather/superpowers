---
name: requesting-code-review
description: 在完成任务、实现主要功能或合并前使用，以验证工作满足需求
---

# 请求代码评审

分派一个代码评审子代理，在问题级联之前捕获它们。评审者得到为评估精确构造的上下文——绝不要你会话的历史。这让评审者聚焦于工作产物，而非你的思考过程，并为你继续工作保留上下文。

**核心原则：** 尽早评审，频繁评审。

## 何时请求评审

**强制：**
- 子代理驱动开发中每个任务之后
- 完成主要功能之后
- 合并到 main 之前

**可选但有价值：**
- 卡住时（新鲜视角）
- 重构之前（基线检查）
- 修复复杂 bug 之后

## 如何请求

**1. 获取 git SHA：**
```bash
BASE_SHA=$(git rev-parse HEAD~1)  # 或 origin/main
HEAD_SHA=$(git rev-parse HEAD)
```

**2. 分派代码评审子代理：**

分派一个 `general-purpose` 子代理，填写 [code-reviewer.md](code-reviewer.md) 中的模板

**占位符：**
- `{DESCRIPTION}` - 你构建的内容的简短摘要
- `{PLAN_OR_REQUIREMENTS}` - 它应该做什么
- `{BASE_SHA}` - 起始提交
- `{HEAD_SHA}` - 结束提交

**3. 据反馈行动：**
- 立即修 Critical 问题
- 在继续前修 Important 问题
- 记下 Minor 问题留待以后
- 如果评审者错了，反驳（附理由）

## 示例

```
[刚完成任务 2：添加验证函数]

你：让我在继续之前请求代码评审。

BASE_SHA=$(git log --oneline | grep "Task 1" | head -1 | awk '{print $1}')
HEAD_SHA=$(git rev-parse HEAD)

[分派代码评审子代理]
  DESCRIPTION: Added verifyIndex() and repairIndex() with 4 issue types
  PLAN_OR_REQUIREMENTS: Task 2 from docs/superpowers/plans/deployment-plan.md
  BASE_SHA: a7981ec
  HEAD_SHA: 3df7661

[子代理返回]：
  优点：干净的架构，真实测试
  问题：
    Important：缺少进度指示器
    Minor：上报间隔的魔法数字（100）
  评估：可以继续

你：[修复进度指示器]
[继续任务 3]
```

## 与工作流集成

**子代理驱动开发：**
- 每个任务之后评审
- 在问题复合之前捕获
- 进入下一个任务前修复

**executing-plans：**
- 每个任务之后或自然检查点评审
- 获取反馈、应用、继续

**临时开发：**
- 合并前评审
- 卡住时评审

## 红旗

**绝不：**
- 因为"简单"就跳过评审
- 忽略 Critical 问题
- 带着未修复的 Important 问题继续
- 与有效的技术反馈争辩

**如果评审者错了：**
- 用技术推理反驳
- 展示证明它能工作的代码/测试
- 请求澄清

模板见：[code-reviewer.md](code-reviewer.md)
