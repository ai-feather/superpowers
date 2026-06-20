---
name: executing-plans
description: 当你有一份写好的实现计划，要在单独会话中带评审检查点执行时使用
---

# 执行计划

## 概览

加载计划，批判性评审，执行所有任务，完成时报告。

**开始时宣告：** "我在用 executing-plans 技能来实现这个计划。"

**注意：** 告诉你的搭档，Superpowers 在能访问子代理时工作得好得多。如果在支持子代理的平台上运行，其工作质量会显著更高（Claude Code、Codex CLI、Codex App、Copilot CLI 和 Gemini CLI 都符合；见 `../using-superpowers/references/` 里的按平台工具参考）。如果子代理可用，用 superpowers:subagent-driven-development 而非此技能。

## 流程

### 步骤 1：加载并评审计划
1. 读计划文件
2. 批判性评审——识别对计划的任何问题或疑虑
3. 如有疑虑：在开始前向你的搭档提出
4. 如无疑虑：为计划项创建待办并继续

### 步骤 2：执行任务

对每个任务：
1. 标记为 in_progress
2. 精确遵循每一步（计划有细粒度步骤）
3. 按指定运行验证
4. 标记为 completed

### 步骤 3：完成开发

所有任务完成并验证后：
- 宣告："我在用 finishing-a-development-branch 技能来完成这项工作。"
- **必需子技能：** 用 superpowers:finishing-a-development-branch
- 遵循该技能来验证测试、呈现选项、执行选择

## 何时停下并寻求帮助

**立即停止执行，当：**
- 遇到阻塞（缺依赖、测试失败、指令不清）
- 计划有关键缺口阻碍开始
- 你不理解某条指令
- 验证反复失败

**请求澄清而非猜测。**

## 何时回到更早步骤

**回到评审（步骤 1），当：**
- 搭档根据你的反馈更新了计划
- 根本方法需要重新思考

**不要硬闯阻塞**——停下并询问。

## 记住
- 先批判性评审计划
- 精确遵循计划步骤
- 不要跳过验证
- 计划要求时引用技能
- 阻塞时停下，不要猜
- 未经用户明确同意，绝不在 main/master 分支上开始实现

## 集成

**必需的工作流技能：**
- **superpowers:using-git-worktrees** - 确保隔离工作区（创建一个或验证既有的）
- **superpowers:writing-plans** - 创建本技能执行的计划
- **superpowers:finishing-a-development-branch** - 所有任务完成后完成开发
