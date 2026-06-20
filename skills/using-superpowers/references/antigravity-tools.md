# Antigravity CLI（`agy`）工具映射

技能用动作说话（"分派一个子代理"、"创建一个待办"、"读一个文件"）。在 Antigravity CLI（`agy`）上，这些解析为下面的工具。

| 技能请求的动作 | Antigravity CLI 等价物 |
|----------------------|----------------------|
| 读文件 | `view_file` |
| 创建新文件 | `write_to_file` |
| 编辑文件 | `replace_file_content` |
| 一次编辑文件的多个位置 | `multi_replace_file_content` |
| 运行 shell 命令 | `run_command` |
| 搜索文件内容 | `grep_search` |
| 按名查找文件 / 列出目录 | `list_dir`（没有专门的 glob 工具——把 `list_dir` 与 `grep_search` 结合） |
| 抓取 URL | `read_url_content` |
| 搜索网络 | `search_web` |
| 向你的搭档提出结构化问题 | `ask_question` |
| 分派子代理（`Subagent (general-purpose):` 模板） | `invoke_subagent` 配一个内置 `TypeName`——`self` 用于全能力工作，`research` 用于只读（见[子代理支持](#子代理支持)） |
| 多个并行分派 | 在一个 `invoke_subagent` 调用的 `Subagents` 数组里多个条目 |
| 任务跟踪（"创建待办"、"标记完成"） | 一个 **task artifact**——`write_to_file` 配 `IsArtifact: true` 和 `ArtifactType: "task"`（见[任务跟踪](#任务跟踪)）。**不是** `manage_task`，它管理后台进程。 |

## 调用技能——读它的 `SKILL.md`

Antigravity 在每个会话开始时把每个已安装技能的 `name` + `description` 呈现给你，但它**没有 `Skill`/`activate_skill` 工具**。要加载技能，**用 `view_file` 读它的 `SKILL.md`，当技能适用时设置 `IsSkillFile: true`**——例如对
`.../plugins/superpowers/skills/<skill-name>/SKILL.md` 调用 `view_file` 并设 `IsSkillFile: true`。
（`IsSkillFile` 是 agy 自己的信号，表示你在读文件以*执行其指令*，而非编辑或预览它——每当你加载技能时设置它。）

这是本 harness 上受祝福的技能加载机制。通用规则"绝不要手动读技能文件"意思是"不要绕过你平台的技能加载机制"——而在 Antigravity 上，读 `SKILL.md` *就是*那个机制。读它是在遵循规则，而非破坏它。

你已经知道哪些技能存在以及它们的用途：它们的名字和描述在会话开始时就在你面前。当一个描述匹配你即将做的事时，在行动前读那个技能的 `SKILL.md`。

## 子代理支持

Antigravity 用 `invoke_subagent` 分派子代理，在 `Subagents` 数组里传每个一个 `TypeName`。两个 `TypeName` 是**内置**的——直接用，无需 `define_subagent`：

- **`self`** —— 你的完整克隆，拥有你的每个工具（包括 `write_to_file`/`replace_file_content`/`run_command`）。通用工作的安全默认：实现、修复、任何编辑文件或运行命令的事。
- **`research`** —— 只读（文件读取、`grep_search`、网络/URL 抓取；无写入或命令访问）。当你特别想要一个不能改动的子代理时用它——调查和只读评审。

只在需要自定义系统提示词或能力组合时调用 `define_subagent`：设 `enable_write_tools: true` 以授予文件编辑**和** `run_command`，`enable_subagent_tools` 用于嵌套分派，`enable_mcp_tools` 用于 MCP。然后用你给的名字调用它。（`manage_subagents` 列出/杀死运行中的子代理。）

技能用 `Subagent (general-purpose):` 分派，要么引用一个提示词模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`），要么提供内联提示词。在 Antigravity 上：

| 技能分派形式 | Antigravity 等价物 |
|---------------------|----------------------|
| 实现者风格的 `*-prompt.md` 模板（写代码、跑测试） | 填充模板，然后用 `TypeName: "self"` 调用 `invoke_subagent` 并带填充后的提示词 |
| 只读评审者模板（`task-reviewer`、`code-reviewer`、`requesting-code-review` 的 `./code-reviewer.md`） | 用 `TypeName: "research"` 调用 `invoke_subagent` 并带填充后的评审模板 |
| 内联提示词（未引用模板） | 用 `TypeName: "self"`（或任务只读时用 `"research"`）调用 `invoke_subagent` 并带你的内联提示词 |

### 提示词填充

技能提供带占位符的提示词模板，如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]`。在把完整提示词传给 `invoke_subagent` 之前填充所有占位符。提示词模板本身包含代理的角色、评审标准和期望的输出格式——子代理会遵循它。

### 并行分派

把多个条目放进单个 `invoke_subagent` 调用的 `Subagents` 数组以并行运行独立的子代理工作。保持依赖任务顺序，但不要为了保留更简单的历史而把独立的子代理任务串行化。

## 任务跟踪

Antigravity **没有待办 / `TodoWrite` 工具**（`manage_task` 管理后台进程——`list`/`kill`/`status`/`send_input`——它*不是*清单）。当技能说创建待办列表或跟踪任务时，维护一个 **task artifact**：一个用 `write_to_file` 保存的 markdown 清单（`IsArtifact: true`、`ArtifactMetadata.ArtifactType: "task"`），随着推进用 `replace_file_content` / `multi_replace_file_content` 编辑。

在任何多步任务开始时，创建 task artifact，列出你计划的每一步。完成每步后，编辑 artifact 标记完成（`- [x]`）。如果计划变了，更新清单。保持它最新——它是剩余工作的真相来源；一旦对话变长，在开始每步前重读它。
