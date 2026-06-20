# Superpowers

Superpowers 是一套面向编码代理的完整软件开发方法论，构建在一组可组合的技能（skill）之上，并通过一些初始指令确保你的代理会使用它们。


## 我们在招聘！

我们正在招聘一位全职人员，协助处理 Superpowers 的社区和代码工作。
岗位详情请见 https://primeradiant.com/jobs/superpowers-community-engineer/
如果你觉得身边有合适的人选，欢迎向我们推荐。

## 快速开始

为你的代理赋予 Superpowers：[Claude Code](#claude-code)、[Antigravity](#antigravity)、[Codex App](#codex-app)、[Codex CLI](#codex-cli)、[Cursor](#cursor)、[Factory Droid](#factory-droid)、[Gemini CLI](#gemini-cli)、[GitHub Copilot CLI](#github-copilot-cli)、[Kimi Code](#kimi-code)、[OpenCode](#opencode)、[Pi](#pi)。

## 工作原理

从你启动编码代理的那一刻开始。一旦它察觉你在构建东西，它*不会*立刻跳进去写代码，而是退后一步，问你到底想做什么。

当它通过对话梳理出一份规格（spec）后，会分块展示给你，每块都足够短，便于你真正阅读和消化。

在你签字认可设计方案之后，代理会整理出一份实施计划（plan），清晰到连一个热情高涨但品味糟糕、毫无判断力、毫无项目背景、还抗拒测试的初级工程师也能照着做。它强调真正的红/绿 TDD、YAGNI（You Aren't Gonna Need It）和 DRY。

接下来，当你说出 "go"，它会启动一个*subagent-driven-development*（子代理驱动开发）流程，让若干代理依次完成每个工程任务、检查并审查它们的工作，然后继续推进。代理常常会自主工作数小时而不偏离你们共同制定的计划。

里面还有很多内容，但这套系统的核心就在这里。而且由于技能会自动触发，你不需要做任何额外的事情。你的编码代理天然就拥有 Superpowers。

## 商业服务

如果你在企业环境中使用 Superpowers，并且希望获得商业支持、额外工具或托管式消费管理，欢迎随时通过 sales@primeradiant.com 与我们联系。

## 安装

安装方式因宿主（harness）而异。如果你同时使用多个宿主，需要分别为每个宿主单独安装 Superpowers。

### Claude Code

Superpowers 可通过 [Claude 官方插件市场](https://claude.com/plugins/superpowers)获取。

#### 官方市场

- 从 Anthropic 官方市场安装插件：

  ```bash
  /plugin install superpowers@claude-plugins-official
  ```

#### Superpowers 市场

Superpowers 市场为 Claude Code 提供 Superpowers 以及其他一些相关插件。

- 注册市场：

  ```bash
  /plugin marketplace add obra/superpowers-marketplace
  ```

- 从该市场安装插件：

  ```bash
  /plugin install superpowers@superpowers-marketplace
  ```

### Antigravity

通过本仓库把 Superpowers 作为插件安装：

```bash
agy plugin install https://github.com/obra/superpowers
```

Antigravity 会运行该插件的 session-start hook，因此 Superpowers 从第一条消息起就处于激活状态。使用相同命令重新安装即可更新。

### Codex App

Superpowers 可通过 [Codex 官方插件市场](https://github.com/openai/plugins)获取。

- 在 Codex 应用中，点击侧边栏的 Plugins。
- 你应该能在 Coding 区看到 `Superpowers`。
- 点击 Superpowers 旁边的 `+`，并按提示操作。

### Codex CLI

Superpowers 可通过 [Codex 官方插件市场](https://github.com/openai/plugins)获取。

- 打开插件搜索界面：

  ```bash
  /plugins
  ```

- 搜索 Superpowers：

  ```bash
  superpowers
  ```

- 选择 `Install Plugin`。

### Cursor

- 在 Cursor Agent 聊天中，从市场安装：

  ```text
  /add-plugin superpowers
  ```

- 或者在插件市场中搜索 "superpowers"。

### Factory Droid

- 注册市场：

  ```bash
  droid plugin marketplace add https://github.com/obra/superpowers
  ```

- 安装插件：

  ```bash
  droid plugin install superpowers@superpowers
  ```

### Gemini CLI

- 安装扩展：

  ```bash
  gemini extensions install https://github.com/obra/superpowers
  ```

- 之后更新：

  ```bash
  gemini extensions update superpowers
  ```

### GitHub Copilot CLI

- 注册市场：

  ```bash
  copilot plugin marketplace add obra/superpowers-marketplace
  ```

- 安装插件：

  ```bash
  copilot plugin install superpowers@superpowers-marketplace
  ```

### Kimi Code

Superpowers 可在 Kimi Code 的插件市场中获取。

- 打开 Kimi Code 的插件管理器：

  ```text
  /plugins
  ```

- 进入 `Marketplace` > `Superpowers` 并安装。

- 或者直接从本仓库安装：

  ```text
  /plugins install https://github.com/obra/superpowers
  ```

- 详细文档：[docs/README.kimi.md](docs/README.kimi.md)

### OpenCode

OpenCode 使用其自己的插件安装机制；即使你已经在其他宿主中使用过 Superpowers，也需要单独安装一次。

- 告诉 OpenCode：

  ```
  Fetch and follow instructions from https://raw.githubusercontent.com/obra/superpowers/refs/heads/main/.opencode/INSTALL.md
  ```

- 详细文档：[docs/README.opencode.md](docs/README.opencode.md)

### Pi

通过本仓库把 Superpowers 作为 Pi 包安装：

```bash
pi install git:github.com/obra/superpowers
```

为本地开发，可以用这个 checkout 作为临时包加载到 Pi 中运行：

```bash
pi -e /path/to/superpowers
```

Pi 包会加载 Superpowers 的技能，以及一个在会话启动时以及压缩（compaction）之后注入 `using-superpowers` 引导逻辑的小型扩展。Pi 原生支持技能，因此不需要兼容性的 `Skill` 工具。子代理和任务列表工具仍然是可选的 Pi 配套包。

## 基本工作流

1. **brainstorming** - 在编写代码之前激活。通过提问细化粗略想法，探索备选方案，分节呈现设计供你确认。保存设计文档。

2. **using-git-worktrees** - 在设计获批后激活。在新分支上创建隔离的工作空间，运行项目初始化，验证干净的测试基线。

3. **writing-plans** - 在设计获批后激活。把工作拆分成小颗粒任务（每个 2-5 分钟）。每个任务都有精确的文件路径、完整的代码和验证步骤。

4. **subagent-driven-development** 或 **executing-plans** - 在拿到计划后激活。为每个任务派发一个全新的子代理，并做两阶段审查（规格符合性，然后是代码质量）；或者分批执行并设置人工检查点。

5. **test-driven-development** - 在实现阶段激活。强制执行 RED-GREEN-REFACTOR：先写失败的测试，看它失败，写最小代码，看它通过，提交。删除在测试之前写出的代码。

6. **requesting-code-review** - 在任务之间激活。对照计划进行审查，按严重程度报告问题。严重问题会阻断进展。

7. **finishing-a-development-branch** - 在任务完成时激活。验证测试，提供选项（合并/PR/保留/丢弃），清理 worktree。

**代理在任何任务之前都会先检查相关技能。** 这是强制性的工作流，不是建议。

## 内部包含什么

### 技能库

**测试**
- **test-driven-development** - RED-GREEN-REFACTOR 循环（包含测试反模式参考）

**调试**
- **systematic-debugging** - 4 阶段根因流程（包含 root-cause-tracing、defense-in-depth、condition-based-waiting 等技术）
- **verification-before-completion** - 确保问题真正被修复

**协作**
- **brainstorming** - 苏格拉底式设计细化
- **writing-plans** - 详细的实施计划
- **executing-plans** - 带检查点的分批执行
- **dispatching-parallel-agents** - 并发的子代理工作流
- **requesting-code-review** - 审查前检查清单
- **receiving-code-review** - 响应反馈
- **using-git-worktrees** - 并行开发分支
- **finishing-a-development-branch** - 合并/PR 决策工作流
- **subagent-driven-development** - 带两阶段审查（规格符合性，然后是代码质量）的快速迭代

**元**
- **writing-skills** - 按照最佳实践创建新技能（包含测试方法学）
- **using-superpowers** - 技能系统入门

## 哲学

- **测试驱动开发** - 永远先写测试
- **系统化胜过临时拍脑袋** - 流程胜过猜测
- **降低复杂度** - 以简洁为首要目标
- **证据胜过声明** - 先验证，再宣布成功

阅读[最初的发布公告](https://blog.fsck.com/2025/10/09/superpowers/)。

## 贡献

下面是 Superpowers 的一般贡献流程。请记住，我们通常不接受新增技能的贡献，而且对技能的任何更新都必须在我们支持的所有编码代理上正常工作。

1. Fork 本仓库
2. 切换到 'dev' 分支
3. 为你的工作创建一个分支
4. 按照 `writing-skills` 技能来创建和测试新的或修改后的技能
5. 提交 PR，并确保完整填写 pull request 模板。

技能行为测试使用来自 [superpowers-evals](https://github.com/prime-radiant-inc/superpowers-evals/) 的 drill 评测工具，需克隆到 `evals/` 下——设置方式参见 `evals/README.md`。插件基础设施测试位于 `tests/`，通过对应的 `run-*.sh` 或 `npm test` 运行。

完整指南见 `skills/writing-skills/SKILL.md`。

## 更新

Superpowers 的更新方式在一定程度上取决于所用的编码代理，但通常是自动的。

## 许可证

MIT 许可证——详见 LICENSE 文件

## 视觉伴侣遥测

由于技能和插件不会向创作者反馈任何信息，我们无从知道你们当中有多少人在使用 Superpowers。默认情况下，brainstorming 的可选视觉伴侣功能中使用的 Prime Radiant logo 会从我们的网站加载。其中包含当前使用的 Superpowers 版本，但不含任何关于你的项目、提示或编码代理的细节。我们看不到你的点击，也看不到你在构建什么。它只是帮助我们大致了解有多少人在使用 Superpowers 以及他们用的是哪个版本。它是 100% 可选的。要禁用它，可将环境变量 `SUPERPOWERS_DISABLE_TELEMETRY` 设为任意真值。Superpowers 也会尊重 Claude Code 的 `DISABLE_TELEMETRY` 和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 退出选项。

## 社区

Superpowers 由 [Jesse Vincent](https://blog.fsck.com) 与 [Prime Radiant](https://primeradiant.com) 的伙伴们共同打造。

- **Discord**：[加入我们](https://discord.gg/35wsABTejz)，获取社区支持、提出问题，并分享你在用 Superpowers 构建什么
- **Issues**：https://github.com/obra/superpowers/issues
- **发布公告**：[订阅](https://primeradiant.com/superpowers/)以获得新版本通知
