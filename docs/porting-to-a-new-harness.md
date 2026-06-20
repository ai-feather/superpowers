# 将 Superpowers 移植到新的宿主

本指南讲解如何为新宿主——也就是那些不是 Claude Code 的 IDE、CLI 或代理运行器——添加支持，使 Superpowers 技能在其上能够像原生环境一样自动触发。

本指南分为两层。**第 1–3 部分**讲解系统的工作原理，以及如何判断一个宿主到底能不能被支持；在动手之前请先读完这些内容。**第 4–8 部分**是为代理（由你的搭档监督执行）准备的一份可执行流程，用于端到端完成移植直至发布。附录索引了当前的参考实现，方便你直接复制最接近的那一个。

不同宿主的集成机制各不相同，而且还会持续变化。本指南刻意只讲那些**不变量**——即无论机制如何都必须成立的条件——并指引你去看一个活跃的参考实现去复制。当本指南与代码不一致时，以代码为准；同时请修正本指南。

## 开始之前

添加新宿主是本仓库中风险最高的贡献类型。在写任何东西之前：

- 完整阅读 `CLAUDE.md` 和 `.github/PULL_REQUEST_TEMPLATE.md`——贡献者规则和新宿主 PR 的要求都不是可选项。
- 在 open **和 closed** 的 PR 中搜索是否有人曾为这个宿主做过尝试。如果有，先搞清楚它为何停滞，再开始你自己的工作。

---

## 第 1 部分 —— Superpowers 如何跨宿主工作

Superpowers 在任何地方都是同一份内容。不同宿主之间变化的，只是那层把内容投递给模型、并把指令翻译成宿主原生工具的薄层。组件有三：

1. **技能（与宿主无关）。** `skills/` 下的所有内容都是真相来源，由每个宿主原样共享。技能描述的是*动作*——“调用一个技能”、“读一个文件”、“派发一个子代理”、“创建一个 todo”——从不点名某个具体工具。正因如此，同一个技能正文才能在 Claude Code、Codex、Gemini、pi 和其他宿主上不经修改地运行。

2. **工具映射（每个宿主各自一份）。** 每个宿主都需要把这套动作词汇表翻译成它真实的工具名。这份翻译放在 `skills/using-superpowers/references/<harness>-tools.md` 里，和/或内联在宿主的引导注入器中（见第 5 部分）。比如它会写：“*派发一个子代理* → 调用 `task` 并传 `subagent_type`。”

3. **引导（每个宿主各自一份）。** 在每次会话开始时，完整的 `skills/using-superpowers/SKILL.md` 会被注入到模型上下文中，外面包一层 `<EXTREMELY_IMPORTANT>` 标签，并附上工具映射。正是这个被注入的技能教会模型：技能这种东西存在，并且它在行动前必须检查是否有相关技能可用。**引导就是整个集成。** 没有它，技能文件是死的——躺在磁盘上，永远不会被调用。

### 让这一切成立的两条规则

**1. 技能描述的是动作，不是工具。** **不要**为了让技能适配你的宿主去修改技能正文。移植只会新增一个工具映射参考文件和一个引导注入器，绝不深入 `skills/*/SKILL.md` 里去替换工具名。（项目贡献者指南把技能内容当作精心调校的行为塑造代码；以“合规”为由重写它会被直接拒掉。）

**2. 一切都通过宿主自己的安装机制发布。绝不编辑用户的文件。** 引导、技能、工具映射，全都作为*宿主所安装内容的一部分*投递——一个插件、一个扩展、一个市场条目、一个随扩展打包的上下文文件。移植**绝不可以**去碰用户的全局或个人配置（`~/.gemini/config/AGENTS.md`、`settings.json`、`trustedFolders.json`、手改的 `~/.bashrc` 等）来注入任何东西。宿主拥有它所加载的内容；你的安装产物是唯一允许写入的东西。如果安装机制确实无法承载引导，那是一个需要上报的限制（第 6 部分）——绝不意味着可以手改用户的配置。（形态 C *不是*例外：Gemini 的上下文文件之所以可以，是因为它*随已安装的扩展一同发布*，并且由 manifest 的 `contextFileName` 声明——宿主加载的是扩展自己的文件，不是你在用户 home 下编辑的文件。）

---

## 第 2 部分 —— 这个宿主能不能被支持？

一个宿主只有能满足下面全部条件，才能支持 Superpowers。写代码之前先检查这些——如果第一条不过，就停下。

### 硬性要求：会话开始时自动注入

宿主必须允许你**在每次会话开始时，无需你的搭档逐会话地选择启用**，就把文本注入到模型上下文中。这是唯一一个不可协商的能力。它可以是任何形式：

- 一个 **hook/事件系统**，在会话开始时运行 shell 命令并读取其 stdout（Claude Code、Codex、Cursor、Copilot CLI），或
- 一个 **进程内插件/扩展**，带有会话开始或消息生命周期回调，可以改写消息数组（OpenCode、pi），或
- 一种 **指令文件**约定，宿主会加载一个*由你已安装的扩展发布并声明*的上下文文件（例如 Gemini 的 `contextFileName` 指向扩展自带的 `GEMINI.md`）——而不是你在用户 home 下编辑的文件。

如果让 Superpowers 出现在模型面前的唯一方式，是要求你的搭档每次会话都手动启用一次（粘贴一段 prompt、运行一条命令、打开某个模式），那这个宿主就**无法**被妥善支持。第 3 部分的验收测试会失败，PR 会被关掉。这是“移植”不是真正移植的最常见原因。

### 其余能力清单

| 能力 | 为什么需要 | 缺失时 |
|---|---|---|
| **技能发现 + 调用** | 模型必须能按需加载某个技能的完整内容 | 如果没有原生技能工具，被认可的后备方案是直接 `read` 对应的 `SKILL.md`——见第 5 部分。一个既没有技能工具也不能读文件的宿主无法工作。 |
| **文件读 / 写 / 编辑** | 几乎每个技能都会操作文件 | 必需。无可行替代方案。 |
| **运行 shell 命令** | TDD、验证、git 流程 | 必需。 |
| **子代理 / 任务派发** | `dispatching-parallel-agents`、`subagent-driven-development` | 可降级：如果不可用，这些具体技能会让模型改成在原地完成工作或报告缺失能力——*绝不*可以臆造一次 `Task` 调用。有些宿主把它藏在某个配置开关后面（例如 Codex 需要启用 multi-agent）。 |
| **Todo / 任务跟踪** | 多个技能里的进度跟踪 | 可降级：回退到一个 plan 文件或 `TODO.md`。 |
| **抓取网页 / 搜索** | 少数技能 | 可降级。 |
| **shell 或 polyglot 脚本执行（Windows）** | 仅 shell-hook 形态需要，仅当你想支持 Windows 时 | 见第 7 部分。进程内插件类宿主完全不受此影响。 |

“可降级”的意思是：该技能本身就含有针对该工具缺失时的后备措辞。你在工具映射里的工作，是在工具存在时指向真实工具，在工具不存在时复用那段后备措辞。

### 你可能根本不需要新建目录

有些“新宿主”其实只是换了安装器的现有集成。比如 Factory 的 Droid 通过它自己的 `plugin install` 命令消费 Claude Code 插件，在这里不需要新增任何文件。开始构建之前，先看看这个宿主能不能直接加载某个现有 manifest。一个除了在 README 加一段话之外对本仓库毫无改动的移植，是完全合格的结果。

---

## 第 3 部分 —— 完成的定义

当一个移植**所有**以下条件都成立时，才算完成：

1. `using-superpowers` 引导在每次会话开始时都会加载，每次都加载，无需逐会话启用。
2. 该宿主有一份工具映射（放在 `references/<harness>-tools.md`、内联在引导中，或两者兼有——见第 5 部分）。
3. 技能确实能被调用——原生调用，或通过文档记载的读 `SKILL.md` 后备方案——并且模型会遵循它们。
4. **验收测试通过。** 在一次干净会话中，用户消息：

   > Let's make a react todo list

   会在*写任何代码之前*自动触发 `brainstorming` 技能。抓取完整 transcript——PR 要求附上它。
5. 测试覆盖该集成（第 5 部分）并且通过。
6. 一个真实用户能够通过宿主自己的机制安装它（不是靠手动复制文件），并且在适用时版本号被记录到 `.version-bump.json`（第 6 部分）。注意有些安装器会在安装时改写或剥掉 manifest（有的会把它削成只剩 `{"name": …}`），所以“*已安装的*文件报告仓库版本”并不总能做到——以源 manifest 处的版本跟踪为准，不要把被改写过的已安装 manifest 当作失败。

完整验收测试之前的一个快速烟雾检查：开一个会话，让模型描述它有哪些 superpowers。如果引导注入成功，它会知道它有这些能力。（OpenCode 的安装文档用 `opencode run --print-logs "hello" 2>&1 | grep -i superpowers` 通过另一种机制——日志 grep 而不是问模型——达到同样目的；`2>&1` 很关键，因为日志走 stderr。）找到你宿主的等价做法。

---

## 第 4 部分 —— 选定你的集成形态

共有三种结构形态，区别在于*你如何把引导推到模型面前*。挑出与你宿主暴露能力匹配的那一种，然后复制对应的参考实现。这个形态几乎决定了第 5 部分里的一切——下面的步骤会根据它分叉。

### 怎么判断你属于哪种形态

在选择路由之前，要先弄清宿主的*真实*机制——不要假设它文档完备，也不要假设它和它 fork 自的那个宿主行为一致。

**找到表面：**

- **在网上搜索该宿主的文档**（extension / plugin / hook / skill / MCP / “context file” / “rules file”）。厂商工具变化很快，搜索胜过依赖训练知识。
- **找到并阅读一个已有的第三方扩展/插件。** 一个真实可用的例子胜过文档——它展示了 manifest 形态、安装命令，以及宿主到底加载哪些组件。
- 查看宿主在启动时加载什么：一个 settings 文件？一个扩展目录？一个按项目或全局的指令文件（`AGENTS.md`、`<NAME>.md`）？

**如果文档不全，就经验性地逆向它**（真正的移植者每一条都做过）：

- 对二进制跑 `strings` / 在安装目录里 grep hook 事件名、配置路径，以及它读取的指令文件名。
- **让正在运行的模型枚举它自己的工具名**——例如“列出你能调用的每个工具的精确机器名，每行一个”。这是不靠臆造而获得工具名的权威方式（见第 4 步）。
- 用**唯一标记测试**验证每一个假设：通过你认为可行的机制注入一段胡言乱语的 token，开一个全新会话，确认这个 token 确实到达了模型。

**一个 fork 不会继承父项目的行为。** 一个派生自其他宿主（例如某个派生自 Gemini 的 CLI）的宿主，可能暴露父项目的 manifest 字段和 `@`-include 语法，却*并不以同样方式兑现它们*。用标记验证；绝不要假设父项目的配方能照搬。

然后路由到一种形态：

- 会话开始时运行一条 shell 命令并读取其 stdout → **形态 A**。
- 一个带生命周期回调、让你在其中跑代码的插件/扩展模块 → **形态 B**。
- 永远只有一个常开的指令文件，没有 hook 也没有代码插件 → **形态 C**。

**各形态可组合——它们并非互斥。** *技能发现*机制和*引导*机制不一定是同一种形态——但**两者都必须仍然依附于安装机制**（规则 2）。分开回答两个问题：*技能在哪里被发现？* 以及 *引导如何每次会话都到达模型？* 一个宿主可能通过插件安装技能，却需要引导以另一种随安装发布的方式投递（一个由扩展声明的上下文文件，或者——见下文——宿主在会话开始时把已安装的 `using-superpowers` 技能自身的 description 呈现出来）。如果有多个安装机制表面都能自动注入，选最可靠的那个。你**不可以**做的，是通过编辑用户全局配置来填补缺口。

### 形态 A —— Shell-hook

宿主有一个 hook 系统，在会话开始时运行一条 shell 命令并从其 stdout 读取 JSON。配置好的命令运行 `run-hook.cmd`——一个 polyglot 包装器，它只负责找到 bash 并分发到指定的脚本；脚本（`hooks/session-start`，或某个宿主专属变体如 `hooks/session-start-codex`）读取 `using-superpowers/SKILL.md` 并打印一个 JSON 对象，其**字段名和嵌套结构每个宿主都不同**。

- 参考：`hooks/session-start`（以及 `hooks/session-start-codex`）、`hooks/run-hook.cmd`，以及各宿主的 hook 配置 `hooks/hooks.json`（Claude Code）、`hooks/hooks-codex.json`（Codex）、`hooks/hooks-cursor.json`（Cursor）。
- Manifest：`.codex-plugin/plugin.json`、`.cursor-plugin/plugin.json` 把宿主指向 `./skills/` 和正确的 `hooks-*.json`。（Claude Code 的 `.claude-plugin/plugin.json` 两个字段都不设——它按约定自动发现 `skills/` 和 `hooks/hooks.json`。）

> **一个 hook *系统* 不等于一个 session-start *事件*。** 一个宿主可能有 `hooks.json` 机制——甚至其二进制里包含字面字符串 `SessionStart`——却没有一个会在会话开始时触发、并能注入上下文的 hook 事件。（某个真实宿主只暴露 pre/post-tool 和 stop 事件；那些 `SessionStart` 字符串是遥测用的。）在押注形态 A 之前，确认你需要的*具体事件*确实存在并能写入模型上下文。如果做不到，引导就该走指令文件（形态 C）。

### 形态 B —— 进程内插件 / 扩展

宿主加载一个 JS/TS 模块，模块暴露生命周期回调。你通过宿主的 API 注册技能目录，并在代码里改写消息数组来注入引导。

- 参考：`.opencode/plugins/superpowers.js`（JavaScript）和 `.pi/extensions/superpowers.ts`（TypeScript）。对于任何**没有原生技能工具**的宿主，pi 是最贴近的参考。

### 形态 C —— 指令文件

宿主既没有 shell hook 也没有代码插件——它的会话开始入口是一个上下文文件，*由你已安装的扩展发布并由 manifest 声明*（例如 Gemini 的 `contextFileName` → 扩展自带的 `GEMINI.md`）。你跑不了代码，也改不了消息；扩展的上下文文件指向引导。这里没有任何注入器去拼装字符串或剥掉 frontmatter——宿主原样加载被引用的内容。**这之所以可行，仅仅是因为该文件是已安装扩展的一部分**——绝不要用“编辑用户全局 `GEMINI.md`/`AGENTS.md`”来替代发布你自己的文件（规则 2）。

- 参考：`gemini-extension.json`（manifest，含 `contextFileName`）、`GEMINI.md`（两个 `@`-include——引导技能和工具映射参考）、`skills/using-superpowers/references/gemini-tools.md`。
- 注意：`@`-include 是 Gemini 的特性。如果你的宿主加载指令文件但没有 include 语法，你必须把引导内容内联到该文件里。
- **不要相信 `@`-include 一定会被展开——要证明它。** 一个派生自 Gemini 的宿主可能接受 `@./path` 语法，却把它当作*模型可选择去读的提示*（它发出一次文件读工具调用），而不是有保证的内联展开。这正是“引导每次会话都可靠在场”和“模型可能去读它”之间的差别。跑一次唯一标记测试：如果该标记不在上下文里且*没有*伴随工具调用，就**把内容内联**而不是用 `@`-include。

### 路由表

| 如果宿主…… | 使用形态 | 复制自 |
|---|---|---|
| 在会话开始时运行 shell 命令并读取其 stdout | A（shell-hook） | Codex（`hooks/session-start-codex` + `hooks/hooks-codex.json` + `.codex-plugin/`） |
| 是一个带 session/message 生命周期回调的 JS/TS 插件宿主 | B（进程内） | OpenCode（`.opencode/`）——或 pi（`.pi/`），如果它没有原生技能工具 |
| 发布一个由扩展声明、且总会被加载的上下文文件 | C（指令文件） | Gemini（`gemini-extension.json` + `GEMINI.md` + `references/gemini-tools.md`） |
| 有 plugin install 命令，且 manifest 有一个会被安装器保留的 `contextFileName`（或等价物） | 通过插件安装器的 C | Antigravity（`.antigravity-plugin/`——`agy plugin install` 会发布一个生成的上下文文件；验证安装器是否保留它——第 6 部分） |

大多数真实宿主都能干净地归到某一行；最后一行是混合情形（规则 2 仍然成立——引导依附于安装机制，绝不是用户配置的编辑）。

---

## 第 5 部分 —— 移植流程

### 第 1 步 —— 研究最贴近的参考实现

打开第 4 部分里针对你形态列出的那些文件，从头到尾读一遍。下面的要点只是摘要；代码才是规格。

### 第 2 步 —— 创建 manifest / 入口点

创建宿主用来识别插件的那些文件。在精神上与现有保持一致：

- **形态 A：** 一个 `*-plugin/plugin.json`（见 `.codex-plugin/plugin.json`），含 `name`、`version`、`description`、author/license/keywords、`"skills": "./skills/"`、`"hooks": "./hooks/hooks-<harness>.json"`。再加 `hooks-<harness>.json` 本身，注册一个 session-start hook，其命令调用 `run-hook.cmd`。
- **形态 B：** 宿主加载的模块（例如 `.<harness>/plugins/*.js`）以及让它能被发现所需的任何 package 元数据。提交进 git 的 package 元数据是**仓库根目录的 `package.json`**：`main` 指向 OpenCode 插件，`pi` 字段（`pi.extensions`、`pi.skills`）加上 `pi-package` keyword 声明了 pi 扩展。各宿主的本地 manifest 和 lockfile 都不进 git——`.opencode/.gitignore` 排除了 `node_modules`、`package.json` 和 lockfile。对你的宿主*本地*安装产物也照此办理，免得污染仓库——但绝不要 gitignore 仓库根目录的 `package.json`，那是被跟踪的真相来源。
  - **构建/依赖检查。** 弄清宿主如何加载你的模块：它是直接跑源码（pi 的 `.ts` 被 `package.json` 原样引用；OpenCode 发布纯 `.js`），还是需要一个转译/构建步骤？Superpowers 是零运行时依赖的。pi 的 `import type { ExtensionAPI }` 之所以能用，恰恰是因为宿主直接跑 `.ts`、在加载时提供该类型，而仓库从不在 CI 里对它做类型检查——这个 import 甚至没有被声明为依赖。如果*你的*宿主真的会对插件做类型检查或打包，那就会出问题：未声明的类型 import 会失败，而 PR 规则为新宿主开脱的仅限于*运行时*依赖，不包括 dev/类型包。遇到这种情况，与维护者确认方案，而不是悄悄加一个依赖。把任何构建产物排除在 git 之外，并文档化构建命令。
- **形态 C（指令文件）：** 一个小 manifest（见 `gemini-extension.json`：`name`、`description`、`version`、`contextFileName`）加上上下文文件本身（`GEMINI.md` 只是两个 `@`-include：引导技能和工具映射参考）。Gemini manifest 没有 `skills` 字段——Gemini 会自动发现已安装扩展中打包的 `skills/` 目录。如果你的宿主有原生技能工具却没有 manifest 字段来注册该目录，你必须找到它的发现约定（读它的扩展文档），然后经验性地验证：接好线之后，让模型列出它可用的技能——如果打包进去的技能不出现，就说明发现机制还没起作用。

### 第 3 步 —— 接好引导注入

这是移植的核心。共同目标：在会话开始时，把 `using-superpowers` 技能内容（外面包一层 `<EXTREMELY_IMPORTANT>` 标签）加上宿主的工具映射推到模型面前，并附注该技能已经处于激活状态，这样模型就不会试图再次加载它。*怎么做*——以及你拼装什么 vs. 宿主原样加载什么——完全取决于你的形态。**不要**把一种形态的配方套到另一种上。

**形态 A —— 一个脚本读 `SKILL.md` 并打印该宿主的 JSON。** 被分发的脚本（`hooks/session-start`）`cat` 整份 `SKILL.md`（包含 frontmatter——没问题；它是原样输出的），在外面套上“You have superpowers… for all other skills use the Skill tool”这一前导，转义它，再打印该宿主的 JSON 形态。形态 A 的工具映射**不**内联在这里——它放在 `references/<harness>-tools.md`（第 4 步）。把 JSON 输出形态弄对。`hooks/session-start` 通过环境变量探测宿主，并打印*三种形态之一*：

- Cursor（设置了 `CURSOR_PLUGIN_ROOT`）：`{ "additional_context": "…" }`
- Claude Code（设置了 `CLAUDE_PLUGIN_ROOT`，未设置 `COPILOT_CLI`）：`{ "hookSpecificOutput": { "hookEventName": "SessionStart", "additionalContext": "…" } }`
- Copilot CLI / SDK 标准（其他情况）：`{ "additionalContext": "…" }`

这是一个坑。发错字段，或多发一个字段，意味着引导要么根本不注入，要么注入两次（Claude Code 同时读取 `additional_context` 和 `hookSpecificOutput` 且不去重，所以两个都发会双重注入）。找到你的宿主期望的精确字段、嵌套和 event-matcher 值。然后决定：给 `hooks/session-start` 加第四个分支，或者——如果该宿主需要不同的引导消息或环境契约——像 Codex 那样加一个专门的 `hooks/session-start-<harness>` 脚本。如果你加分支而你的宿主*也*设置了某个更早分支所 key-on 的环境变量（有些宿主也会设 `CLAUDE_PLUGIN_ROOT`），把你的分支排到那个原本会遮蔽它的分支之前。匹配宿主自己的 event-matcher 字符串（Claude Code 用 `startup|clear|compact`，Codex 用 `startup|resume|clear`，Cursor 用 `sessionStart`）；matcher 错了，hook 会静默不触发。

**hook 配置的 schema 本身也随宿主而异**——不要假设 Claude/Codex 的形态是通用模板。对比 `hooks/hooks.json`、`hooks/hooks-codex.json` 和 `hooks/hooks-cursor.json`：Cursor 的用 `"version": 1`、小写的 `sessionStart` 键、相对路径 `./hooks/run-hook.cmd` 命令，并省略了其他两者使用的 `matcher`/`type`/`async` 字段。让你的 `hooks-<harness>.json` 去匹配最接近的现有文件，而不是某个唯一的规范模板。

hook **命令字符串引用一个由宿主提供的 plugin-root 变量**，它的名字每个宿主都不同：`hooks.json` 用 `${CLAUDE_PLUGIN_ROOT}`，`hooks-codex.json` 用 `${PLUGIN_ROOT}`，Cursor 用相对路径。用你的宿主导出的那个。（`session-start` 脚本自己通过 `dirname` 重新推导出根路径，所以脚本正文不依赖它——但 manifest 里的命令依赖。）

**发现宿主的契约。** 上述三件事——环境变量、JSON 字段/嵌套、matcher 字符串——是宿主的契约，不是 Superpowers 的，所以你必须自己去找来源。读宿主的 hook 文档，或经验性地找：注册一个一次性的 session-start hook，dump 它的环境并输出一个标记，然后观察哪个环境变量能识别该宿主，以及该宿主如何/是否消费你的 stdout。在写真正的分支之前，先把这些钉死。

**形态 B —— 在代码里拼装字符串，然后作为 user 消息注入。** 这里你自己构建引导：读 `SKILL.md`，剥掉它的 YAML frontmatter，拼装 `<EXTREMELY_IMPORTANT>` + 一段简短前导（说明该技能已加载、不可再被调用）+ 剥好的正文 + 内联工具映射 + `</EXTREMELY_IMPORTANT>`。参考实现之间有一个分歧点：OpenCode 的前导写“do NOT use the skill tool…”（假设存在 `skill` 工具），而 pi 的只是写“do not try to load using-superpowers again.”。如果你的宿主没有技能工具，用 pi 的措辞，而不是 OpenCode 的。

把结果作为 **user-role 消息注入，不是 system 消息**——system 消息在每轮重复时会膨胀 token（#750），而且多个 system 消息会破坏某些模型（#894）。有三件事你必须照做：

- **去重守卫。** 生命周期回调可能反复触发（OpenCode 的 transform 在*每个* agent step 上运行；pi 的 `context` 每轮触发）。注入之前，检查是否已存在引导标记，是则跳过。（参考实现选了不同的标记——pi 用一个自定义字符串，OpenCode 用 `EXTREMELY_IMPORTANT` 标签；匹配标签更稳健，因为它不依赖宿主特定的常量。）在模块级别缓存引导内容，免得每次调用都重读、重解析 `SKILL.md`（#1202）。
- **压缩。** 如果宿主会压缩/摘要历史，要在压缩之后重新注入。pi 在 `session_start` 和 `session_compact` 上设一个 `injectBootstrap` 标志，在 `agent_end` 上清掉它，并把消息插在任何开头的压缩摘要消息*之后*。OpenCode 依赖它的逐步重新注入加上去重守卫。
- **message-object 形态是每个宿主都不同的——去发现你自己的，别照抄字面值。** 两个参考用了*互不兼容*的形态：pi 构造 `{ role, content: [{ type, text }], timestamp }`；OpenCode 改写 `message.info.role` 和 `message.parts[]`。从宿主的 API 找出你的消息形态；逐字复制参考的对象字面值会静默失败。

**形态 C —— 把扩展的上下文文件指向引导；什么都不用拼装。** 这里没有注入器，所以你*不*剥 frontmatter，也不构造包好包装的字符串。你的扩展发布的上下文文件（由 manifest 声明——*不是*用户自己的全局文件）拉进两样东西：`using-superpowers` 技能和宿主的工具映射参考。`GEMINI.md` 用两个 `@`-include 做这件事（`@./skills/using-superpowers/SKILL.md` 和 `@./skills/using-superpowers/references/<harness>-tools.md`）；宿主原样加载它们，frontmatter 也照样保留，而 `SKILL.md` 内部本来就带自己的 `<EXTREMELY-IMPORTANT>` 块。如果你的宿主没有 include 语法，就把内容内联到指令文件里。Gemini **不**发布任何“已加载、不要再调用”的前导——对于一个 `@`-include 宿主，内容就是活动的指令集，不是模型会再去加载的技能。如果你发现你的宿主确实会试图再次调用，就在指令文件里加一行字面说明（你没有别的代码途径去加它）。

### 第 4 步 —— 写工具映射

把动作词汇表翻译成宿主的真实工具。覆盖以下每一项动作（只省略那些确实不适用的）：

- 读一个文件
- 创建 / 编辑 / 删除一个文件（一个 `apply_patch` 风格的工具，还是分开的 write/edit？）
- 运行一条 shell 命令
- 搜索文件内容 / 按名字找文件（grep、glob）
- 抓取一个 URL / 网页搜索
- **派发一个子代理**，包括如何传递 agent 类型——以及启用它所需的任何配置开关
- **创建 / 更新 todos**（把较早的 `TodoWrite` 引用当作这个动作处理）
- **调用一个技能** —— 见第 5 步

**从宿主获取真实工具名；绝不臆造。** 如果文档没有列出，权威来源是宿主本身：在活会话里，让模型“列出你能调用的每个工具的精确机器名，每行一个”，用它报告的为准。

**宿主如何找到 `skills/` 目录，本身也是每个宿主都不同**——要确认，不要假设。可能性：一个 manifest 的 `skills` 路径字段（Codex 的 `"skills": "./skills/"`）；一个宿主自动扫描的*同位* `skills/`（在这种情况下路径字段被**忽略**——某个真实宿主只扫描紧挨着 `plugin.json` 的那个 `skills/`）；一次 API/注册调用（OpenCode、pi）；或者你准备一个安装目录，把 manifest 与一个**指向仓库 `skills/` 的符号链接**配对放好，再让安装器指向这个暂存目录（验证安装器会*解引用*符号链接并拷贝真实文件——在依赖它之前用 `agy plugin validate`/`install` 或等价命令确认）。`skills` 路径字段*不*是可移植的。

映射放哪里取决于形态：

- **形态 A：** 放在 `skills/using-superpowers/references/<harness>-tools.md`。代理通过引导触达它——`SKILL.md` 的“Platform Adaptation”一节链接到各宿主的参考文件。（形态 A 宿主没有指令文件；映射*不*内联进 hook 输出。）
- **形态 B：** 映射通常内联进你注入的引导字符串（见 `superpowers.js` 里的 `toolMapping` 常量）。pi 把它放在*两个*地方——内联的 `piToolMapping()` **以及** `references/pi-tools.md`。如果你在两处都维护，两处都要更新，否则移植只完成一半。
- **形态 C：** 放在 `references/<harness>-tools.md`，并把它拉进总是加载的指令文件（例如 `GEMINI.md` `@`-include 了 `gemini-tools.md`）。

你还可以在 `SKILL.md` 的“Platform Adaptation”一节里加一行指向你宿主的指针，让读到引导的代理知道它的映射在哪里。这是一个移植可以对 `SKILL.md` 做的唯一一处编辑——也仅仅因为那一节是指针列表，不是行为塑造内容。它不违反“不要改技能正文”这条规则（第 1 部分）；不要碰任何技能里的其他东西。（这个列表是便利指针，不是穷尽注册表——并不是每个宿主都列出来。）

### 第 5 步 —— 处理没有原生技能工具的宿主

`using-superpowers/SKILL.md` 告诉模型：*绝不手动用文件工具读技能文件——永远使用你平台的技能加载机制。* 这里的重点是“不要绕过机制”，不是“永远不要用文件读”。什么叫“你平台的机制”，取决于宿主——而对于一个没有技能工具的宿主，文档记载的机制*就是*读 `SKILL.md`。所以在那里读它正是遵守规则而不是破坏它。区分三种情况：

1. **原生 `Skill` 风格工具**（Claude Code、Copilot CLI、Gemini 的 `activate_skill`）：把映射指向那个工具。
2. **原生技能*发现*但没有 `Skill` 工具**（pi、Antigravity）：宿主能找到并列出技能，但模型没法调用一个工具去加载某个技能。把技能装到宿主扫描的地方（pi 通过 `resources_discover` → `skillPaths` 注册；OpenCode 通过它的 `config` hook；`agy plugin install` 把它们拷贝进去），并告诉模型：当某技能适用时，**用文件读工具读它的 `SKILL.md`** 来加载——这是这里被认可的方式，正如 `references/pi-tools.md` 所述。

   **对于引导本身，优先用声明的上下文文件（第 6 部分）。** 如果宿主有 `contextFileName` 风格的 manifest 字段——Antigravity 就有——就通过安装器发布一个生成的上下文文件：它一定会被加载，并且同时承载 `using-superpowers` 内容和工具映射。这是强、优先的路径。

   **后备 —— 被呈现出来的技能索引。** 如果没有上下文文件字段，但宿主在每次会话开始时把每个已安装技能的 name + description 呈现出来，你就*既不需要*构建索引*也不需要*运行时列表指令——宿主本身就是索引，而 `using-superpowers` 自身被呈现出来的 description 就可以成为触发模型去加载它的因素。这比声明的上下文文件要弱；相较于上下文文件 / hook / 进程内注入器，它有两件事**给不了**你——要把两件都考虑进去：
   - **它引导的是*触发*，不是*工具映射*。** 一个注入器会在每次会话时把 `<harness>-tools.md` 和 `using-superpowers` 一起前置。这里没有任何东西注入映射——模型只看到技能 *description*，并且必须*读*你的 `references/<harness>-tools.md` 来获取工具名。这之所以能工作，是因为技能描述的是动作（模型在行动时就会去读映射），但它比注入要弱。要确保映射从模型加载的内容里可达——比如从 `SKILL.md` 的 Platform Adaptation 一节链接过去，并与技能一起安装——而不只是躺在仓库里。
   - **没有结构性保证触发一定发生。** 没有 `<EXTREMELY_IMPORTANT>` 包装，没有去重，没有压缩后重新注入——是否触发取决于模型是否选择按它在索引里看到的某个 description 行动。这正是为什么这里验收测试是强制性的：它是*唯一*的保证，所以要在你的用户实际会用的模型上跑它，而不只是最强的那个。
3. **完全没有技能系统：** 没有什么可注册的，*唯一*机制是模型按需读 `SKILL.md`。但模型读不到它找不到的东西：`using-superpowers/SKILL.md` **不**枚举可用的技能，所以只靠它自己，模型不会知道有哪些技能存在或它们的触发条件。你必须提供一条发现路径。两个选项，耐久性不同：(a) 生成一个技能索引（每个 `skills/*/SKILL.md` 的 `name` + `description` frontmatter），把它放在 `<EXTREMELY_IMPORTANT>` 包装内、与工具映射并列（上面的形态 B 配方），这样它就被去重守卫覆盖——但构建期索引会随着新技能增加而过时；或者 (b) 指示模型在运行时列出 `skills/*/SKILL.md` 并读它们的 frontmatter 来找匹配——慢一些但永不过时。除非你有理由不这么做，否则优先 (b)。两者都没有的话，一个无技能系统的移植加载了引导，却静默地从不触发任何其他技能。

在第 2 和第 3 种情况里，在你的工具映射里直白地说读 `SKILL.md` 是受认可的正路，这样模型就不会以为自己在违反“永不读技能文件”的规则。不要在一个没有技能系统的宿主里去找 `skillPaths` 风格的注册 API——第 3 种情况没有这东西。

### 第 6 步 —— 加测试

与现有的各宿主测试风格保持一致：

- **形态 A：** 断言 hook 的 stdout 具备你的宿主消费的精确 JSON 形态，并且包含引导。见 `tests/hooks/test-session-start.sh`，它验证每个宿主的输出形态。
- **形态 B：** 一个单元测试，伪造宿主的插件 API，断言生命周期 handler 已注册、引导只注入一次、去重守卫工作，以及（如果相关）压缩后重新注入工作。见 `tests/pi/test-pi-extension.mjs`。再按 `tests/opencode/` 的风格加一个隔离安装的集成检查。
- 如果引导被缓存，测试当文件缺失时缓存的行为（见 OpenCode 的缓存测试）。

这些自动化测试覆盖的是接线；第 7 步里的 tmux 实跑才是证明集成真的能触发技能的东西。

### 第 7 步 —— 本地安装，然后驱动一个真实实例验证

你无法靠读代码来确认一个移植能用。你必须用你正在进行的移植加载后跑起这个宿主，观察一次真实会话——这也是你产出 PR 所需 transcript 的方式。

**本地安装。** 把宿主的*本地*实例指向你的工作树，而不是某个已发布的构建：

- **形态 A / C：** 从本仓库的本地路径安装插件/扩展（或把它的目录 symlink 到宿主查找的位置）。在它的文档里找到“从本地目录 / git checkout 安装”的路径。
- **形态 B：** 注册本地模块——例如一个指向本地路径的 `opencode.json` `plugin` 条目，或 pi 从仓库解析 `package.json` 字段。

每次改动后都重新安装并重启宿主，因为引导在启动时加载。

**用 tmux 驱动它。** 大多数宿主是交互式 REPL/TUI，没法靠管道 stdin 驱动，所以把宿主放在一个分离的 tmux 会话里跑，用 `send-keys` / `capture-pane` 控制。某个宿主可能宣传一种非交互的“跑一条 prompt”模式（例如 `opencode run "..."`）——可以拿它做快速烟雾检查，但**别依赖它**：这些模式常常不稳定、被鉴权挡住、或被信任检查挡住（某个真实宿主的 `--print` 模式每次都挂住、超时、零输出）。准备好连烟雾检查都*全程*通过 tmux 做。

**先把那些门槛清掉，否则 tmux 会静默卡住。** 许多宿主在首次运行时会卡在 onboarding、“是否信任此目录？”提示、沙箱模式或权限门槛——而一个分离的 tmux 会话会就这么停在那里，不报任何错。开跑之前，预先信任你的临时目录（在宿主的 settings/config 里），或者准备好通过 `send-keys` 回答这些提示，并在你的第一次 `sleep` 里把宿主的启动时间算进去。

```bash
# 1. 分离启动宿主，放进一个用完即弃的项目目录
mkdir -p /tmp/port-smoke
tmux new-session -d -s port-test -c /tmp/port-smoke '<harness-launch-command>'

# 2. 让它初始化——真实 TUI 比你想的慢（10s+ 含模型握手）；调整这个值。
#    然后 capture 并清掉任何挡路的 modal，再敲 prompt：首次 onboarding 和
#    “trust this folder?” 是 modal，在这期间发出的按键是在选菜单项而不是在敲你的 prompt。
sleep 12
tmux capture-pane -t port-test -p          # onboarding / trust prompt? 先用 send-keys 回答它
# (例如 tmux send-keys -t port-test Enter   # 接受 trust prompt——先看清楚再假设)

# 3. 烟雾检查：模型知道它有 superpowers 吗？
#    文本和 Enter 作为分开的 send-keys 发送，中间留一拍——一起发在某些 TUI 上会竞态
#    （Enter 在文本落定之前就到了）。
tmux send-keys -t port-test 'What are your superpowers?'; sleep 0.4; tmux send-keys -t port-test Enter
sleep 5
tmux capture-pane -t port-test -p          # 回复应显示它知道自己的技能

# 4. 验收测试：精确 prompt（注意转义的撇号），全新会话
tmux send-keys -t port-test 'Let'\''s make a react todo list'; sleep 0.4; tmux send-keys -t port-test Enter
# 轮询直到这一轮结束——每隔几秒重新 capture，不要只 capture 一次
sleep 8
tmux capture-pane -t port-test -p          # PASS = brainstorming 在任何代码之前触发

# 5. 保存 transcript 用于 PR，然后清理
tmux capture-pane -t port-test -p > /tmp/port-smoke/transcript.txt
tmux kill-session -t port-test
```

这里会咬人的 tmux 坑：启动后等一下再做第一次 capture；把 prompt 文本和 `Enter` 作为*分开的* `send-keys` 调用，中间留一个短 `sleep`（一起发在某些 TUI 上会竞态），并且 `Enter` 是一个键名而不是 `\n`；agent 的一轮要花时间，所以**在一个循环里 poll `capture-pane`**，而不是只 capture 一次；`capture-pane` 只显示可见面板，所以对于长对话，用宿主自己的 transcript/log 文件作为真相记录；收尾时一定 `kill-session`。

如果烟雾检查显示模型*不*知道它有 superpowers，说明引导没加载——先修这个，再去管验收测试。

---

## 第 6 部分 —— 分发与发布

本仓库里一个能用的集成，要等到真实用户能安装它，才算可用。分发因宿主生态而异——找到你那个：

| 渠道 | 例子 | 你做什么 |
|---|---|---|
| 原生插件市场 | Claude Code | 在 `.claude-plugin/marketplace.json` 注册；用户 `/plugin install`。外部仓库 `superpowers-marketplace` 是用户实际安装的真相来源——见 `CLAUDE.md` 里的发布步骤。 |
| 外部市场 fork，由脚本同步 | Codex | `scripts/sync-to-codex-plugin.sh` 把被跟踪的插件文件 rsync 到一个独立的 fork 仓库并开 PR。读它的 include/exclude 列表，确保你发布的目录树正确（它有意丢掉仓库内部目录和其他宿主的点目录）。 |
| Git-URL 扩展安装 | Gemini、Kimi Code、OpenCode | 用户从一个 git URL 安装（`gemini extensions install …`；Kimi Code `/plugins install …`；一个 `opencode.json` 的 `plugin` 数组条目）。文档化精确命令。 |
| Package-manifest 字段 | pi | 通过仓库根 `package.json` 的字段声明；用户用宿主的 package 命令安装。 |
| 本地安装器（plugin install） | Antigravity（`agy`） | 一个小 `install.sh`，对着一个暂存目录运行宿主自己的 `agy plugin install`，暂存目录里放好 manifest、技能，以及一个生成的 `contextFileName` 上下文文件（即引导）。一切都通过安装机制送达——*不是*靠编辑用户配置（见下文）。 |

然后：

- **一个插件安装器可能静默剥掉*未声明*的文件——所以要把引导做成一个安装器*认得*的文件，绝不能改成编辑用户配置。** `plugin install` 通常只拷贝它认识的组件（skills/agents/commands/mcp/hooks/context），其他一律丢弃，所以一个 manifest 没声明的上下文文件在安装后就消失了。修复办法**不是**放弃并去写用户配置（**规则 2**）——而是把引导声明为一个被认得的组件。按升级顺序：
  - **发布一个 manifest 声明的上下文文件。** 如果宿主有 `contextFileName` 风格的字段（一个扩展声明、每次会话都加载的文件），这是最强、最干净的引导：声明它，安装器既会保留它，*而且*宿主也会加载它。在安装时从活的 `using-superpowers/SKILL.md` + 工具映射（包在 `<EXTREMELY_IMPORTANT>` 里）生成它，这样已安装的引导永不漂移。这正是 `.antigravity-plugin/install.sh` 做的事——`agy plugin install` 会报告 `✔ context : ANTIGRAVITY.md`，而一次干净会话会读 `using-superpowers` 的 SKILL.md、加载 `brainstorming`，并在任何代码之前进入 brainstorming 流程。**用标记验证**安装器保留了文件且宿主加载它：曾经有个移植者错误地结论说做不到，因为他们发布文件时*没有*声明 `contextFileName`，于是它作为未识别文件被剥掉。
  - **否则，就依靠已安装的 `using-superpowers` 技能本身。** 如果宿主在会话开始时把每个已安装技能的 name + description 呈现出来，`using-superpowers` 的 description（“Use when starting any conversation…”）就能提示模型去加载它——安装这个技能*本身*就是引导。更弱（没有保证的包装；它承载触发但不承载工具映射——见第 5 步），所以有声明式上下文文件时优先用它。
  - 如果两者都不行，这个宿主目前还不能被干净地支持——**如实说明**并上报，而不是手改用户配置。

- **写安装文档。** 一个 `docs/README.<harness>.md` 和/或 `.<harness>/INSTALL.md`（见 `docs/README.opencode.md` 和 `.opencode/INSTALL.md`），外加顶层 `README.md` 里的一节安装说明。唯一受支持的安装动作是**运行宿主自己的安装命令**（`agy plugin install`、`gemini extensions install`、`/plugin install` 等）。手动复制技能文件和编辑用户全局/个人配置*都*在禁止之列（规则 2 / PR 规则）。如果宿主根本没有安装命令——它的唯一入口是一个用户拥有的配置文件——那它就过不了“通过安装机制投递”这条规则，你应该上报，而不是发布一个去编辑用户文件的安装器。
- **注册版本。** 如果你的宿主引入了一份*新的*带版本 manifest，把它的路径和版本字段加进 `.version-bump.json`，这样 `scripts/bump-version.sh` 就能让它保持同步（读那个文件看当前跟踪了哪些）。一个没在那里注册的新 manifest 会发布陈旧版本。如果你的宿主骑在一个已被跟踪的文件上——pi 在仓库根 `package.json` 里声明自己，而它已经被列出来了——那就没什么要新增的。
- **如果没有现成渠道合适，你就是在新建一个。** 上面四行可能都不匹配你的宿主。如果它需要 Codex 风格的外部 fork 同步，`scripts/sync-to-codex-plugin.sh` 是要克隆的模板（注意它锚定的 include/exclude 列表和它的 PR 自动化）。并且每当你新增一个按宿主分的目录，都要把它加进*其他*宿主的同步排除项里（例如 `sync-to-codex-plugin.sh` 里的 EXCLUDES 列表），这样你的点目录不会泄漏进它们的分发里。

---

## 第 7 部分 —— 跨平台 / Windows

只与 shell-hook 形态相关。`hooks/run-hook.cmd` 是一个 polyglot：同一个文件既是合法的 Windows 批处理脚本，也是合法的 Unix shell 脚本。在 Windows 上，`cmd.exe` 运行批处理部分，它会找到 `bash`（先是 Git for Windows，然后是 PATH 上的 `bash`）并运行指定的 hook 脚本；如果找不到 bash，它就干净退出，这样宿主仍能工作，只是没有注入。在 Unix 上，开头的 `:` 让批处理块变成 no-op，shell 直接跑脚本。

它强制了两条规则，你也必须遵守：

- **hook 脚本没有扩展名**（`session-start`，不是 `session-start.sh`）。Claude Code 在 Windows 上会对任何包含 `.sh` 的命令前置 `bash`，那会造成二次调用。给你的 hook 脚本起不带扩展名的名字。
- 不要为 hook 脚本写按 OS 区分的变体。一个无扩展名的 bash 脚本加这个 polyglot 包装器就覆盖了全部三个平台。

`hooks/run-hook.cmd` 本身就是权威实现——去读它。背景和分发器模式的设计理由见 `docs/windows/polyglot-hooks.md`。

---

## 第 8 部分 —— 提交 PR

- 目标分支是 **`dev`**。每个 PR 只做一个宿主。
- 填写 PR 模板的 **“New harness support”** 一节，并粘贴完整的验收测试 transcript（那段显示 `brainstorming` 自动触发的“Let's make a react todo list”会话）。没有这份证据的 PR 会被关掉。
- Superpowers 是零依赖插件。不要添加第三方运行时依赖。新增宿主是贡献者规则里唯一允许的例外，即便如此，也只保留集成严格必需的部分——只用于类型、编译后消失的 import 可以；运行时包不行。
- 不要改技能正文（第 1 部分）。如果你发现自己为了移植能跑而去改某个 `SKILL.md`，那修复应该落在你的工具映射里。

---

## 附录 A —— 参考实现（当前）

把这当作活跃索引；有疑问就读文件，不要读这张表。

| 宿主 | 入口点 | 引导机制 | 工具映射 | 测试 | 分发 |
|---|---|---|---|---|---|
| Claude Code | `.claude-plugin/plugin.json` + `hooks/hooks.json` | shell hook → `hooks/session-start`（`hookSpecificOutput.additionalContext`） | 原生 `Skill` 工具；`references/claude-code-tools.md` | `tests/hooks/` | marketplace |
| Codex | `.codex-plugin/plugin.json` + `hooks/hooks-codex.json` | shell hook → `hooks/session-start-codex` | `references/codex-tools.md` | `tests/codex-plugin-sync/`、`tests/hooks/` | fork 同步（`scripts/sync-to-codex-plugin.sh`） |
| Cursor | `.cursor-plugin/plugin.json` + `hooks/hooks-cursor.json` | shell hook → `hooks/session-start`（`additional_context`） | `references/claude-code-tools.md` | `tests/hooks/` | 手工编写 |
| Copilot CLI | （共用 Claude Code 的 hook 路径；`COPILOT_CLI` 环境变量） | shell hook → `hooks/session-start`（`additionalContext`） | `references/copilot-tools.md` | `tests/hooks/` | — |
| Gemini CLI | `gemini-extension.json` + `GEMINI.md` | 指令文件 `@`-include 引导 + 映射 | `references/gemini-tools.md` | — | `gemini extensions install` |
| Kimi Code | `.kimi-plugin/plugin.json` | manifest 的 `sessionStart.skill` 加载 `using-superpowers` | 内联在 manifest 的 `skillInstructions` | `tests/kimi/` | marketplace 或 `/plugins install` GitHub URL |
| OpenCode | `.opencode/plugins/superpowers.js`（通过根 `package.json` 的 `main` 声明） | 进程内：`config` hook 注册技能目录；`experimental.chat.messages.transform` 注入 user 消息 | 内联在 `superpowers.js` | `tests/opencode/` | `opencode.json` 插件 git URL |
| pi | `.pi/extensions/superpowers.ts` | 进程内：`resources_discover` 注册技能；`context` 事件注入 user 消息；带生命周期标志 + 感知压缩 | 内联 `piToolMapping()` **以及** `references/pi-tools.md` | `tests/pi/` | 仓库根 `package.json` 字段 |

## 附录 B —— 咬过移植者的坑

- **需要 opt-in 就不是移植。** 如果你的搭档每次会话都得做点什么才能让 Superpowers 起作用，验收测试就会失败。重读第 2 部分。
- **JSON 字段错了 → 静默失败或双重注入。** 仅形态 A。确认精确的字段/嵌套；Claude Code 不去重地读两个字段。
- **hook 配置的 schema 每个宿主都不同。** 形态 A。Cursor 的 `hooks-cursor.json` 和 Claude/Codex 的看起来毫无相似之处（`version`、小写 `sessionStart`、相对路径命令、无 `matcher`/`type`/`async`）。匹配最接近的现有文件。
- **plugin-root 环境变量每个宿主都不同。** 形态 A。hook 命令用 `${CLAUDE_PLUGIN_ROOT}`（Claude）、`${PLUGIN_ROOT}`（Codex）或相对路径（Cursor）。用你宿主导出的那个；脚本会自己重新推导根路径。
- **system 消息注入。** 形态 B 故意注入一条 *user* 消息（#750、#894）。不要把它“修”成 system 消息。
- **每步 vs 每轮回调。** OpenCode 每步触发（逐次调用去重守卫）；pi 每轮触发（生命周期标志 + `agent_end` 重置）。把一个宿主的去重策略套到另一个的回调频率上，会破坏注入。
- **message-object 形态每个宿主都不同。** 形态 B。pi 和 OpenCode 用互不兼容的形态；去发现你自己的，别复制参考的对象字面值。
- **到处找一个不存在的技能注册 API。** 一个没有技能系统（不只是没有 `Skill` 工具）的宿主没有可注册的东西——模型按需读 `SKILL.md`。不要假设存在 `skillPaths` 等价物。
- **映射在两处。** 对于进程内插件，映射可能既内联又在 `references/` 文件里（pi）。两处都更新。
- **“永不读技能文件”那句话。** 它的意思是“不要绕过你平台的技能加载机制”，不是“永远不用文件读”。在一个无技能工具的宿主上，那个机制*就是*读 `SKILL.md`——在映射里明说这一点（第 5 部分）。
- **Windows 上的 `.sh`。** 保持 hook 脚本无扩展名（第 7 部分）。
- **未注册的版本。** 一个没加到 `.version-bump.json` 的新 manifest 会发布陈旧版本（第 6 部分）。
- **为了让宿主适配而改技能。** 绝不。修复应该落在工具映射里。
