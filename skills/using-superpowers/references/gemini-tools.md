# Gemini CLI 工具映射

技能用动作说话（"分派一个子代理"、"创建一个待办"、"读一个文件"）。在 Gemini CLI 上，这些解析为下面的工具。

| 技能请求的动作 | Gemini CLI 等价物 |
|----------------------|----------------------|
| 读文件 | `read_file` |
| 一次读多个文件 | `read_many_files` |
| 创建新文件 | `write_file` |
| 编辑文件 | `replace` |
| 运行 shell 命令 | `run_shell_command` |
| 搜索文件内容 | `grep_search` |
| 按名查找文件 | `glob` |
| 列出文件和子目录 | `list_directory` |
| 抓取 URL | `web_fetch` |
| 搜索网络 | `google_web_search` |
| 调用技能 | `activate_skill` |
| 分派子代理（`Subagent (general-purpose):` 模板） | `invoke_agent` 配 `agent_name: "generalist"`（可通过 `@generalist` 聊天语法调用——见[子代理支持](#子代理支持)） |
| 多个并行分派 | 在同一响应里多个 `invoke_agent` 调用 |
| 任务跟踪（"创建待办"、"标记完成"） | `write_todos`（状态：pending、in_progress、completed、cancelled、blocked） |

## 指令文件

当技能提到"你的指令文件"时，在 Gemini CLI 上这是 **`GEMINI.md`**。Gemini CLI 分层加载 `GEMINI.md`：全局在 `~/.gemini/GEMINI.md`，项目级文件在工作区目录及其祖先中，子目录的 `GEMINI.md` 文件在某个工具访问那些目录中的文件时加载。

## 个人技能目录

用户级技能位于 **`~/.gemini/skills/`**，**`~/.agents/skills/`** 作为跨运行时别名（与 Codex 和 Copilot CLI 共享）。当两个目录在同一范围同时存在时，`.agents/skills/` 优先。每个技能是一个子目录，包含一个 `SKILL.md`（带 `name` 和 `description` frontmatter）。

## 子代理支持

Gemini CLI 通过 `invoke_agent` 工具分派子代理，它接受 `agent_name` 和 `prompt` 参数。同样的分派也作为聊天语法快捷方式呈现：输入 `@generalist <prompt>` 等价于用 `agent_name: "generalist"` 调用 `invoke_agent`。内置代理名包括 `generalist`、`cli_help`、`codebase_investigator`，以及（启用浏览器工具时）`browser_agent`。

技能用 `Subagent (general-purpose):` 分派，要么引用一个提示词模板文件（例如 `superpowers:subagent-driven-development` 的 `./implementer-prompt.md`），要么提供内联提示词。在 Gemini CLI 上：

| 技能分派形式 | Gemini CLI 等价物 |
|---------------------|----------------------|
| 引用 `*-prompt.md` 模板（实现者、任务评审者、代码评审者等） | 填充模板，然后用 `agent_name: "generalist"` 调用 `invoke_agent` 并带填充后的提示词 |
| 引用 `superpowers:requesting-code-review` 的 `./code-reviewer.md` | 用 `agent_name: "generalist"` 调用 `invoke_agent` 并带填充后的评审模板 |
| 内联提示词（未引用模板） | 用 `agent_name: "generalist"` 调用 `invoke_agent` 并带你的内联提示词 |

### 提示词填充

技能提供带占位符的提示词模板，如 `{WHAT_WAS_IMPLEMENTED}` 或 `[FULL TEXT of task]`。在把完整提示词传给 `invoke_agent` 之前填充所有占位符。提示词模板本身包含代理的角色、评审标准和期望的输出格式——子代理会遵循它。

### 并行分派

Gemini CLI 支持并行子代理分派。在同一个响应里发起多个 `invoke_agent` 调用（或在一个提示词里多个 `@generalist` 调用）以并行运行独立的子代理工作。保持依赖任务顺序，但不要为了保留更简单的历史而把独立的子代理任务串行化。

## 其他 Gemini CLI 工具

这些工具是 Gemini CLI 独有的：

| 工具 | 用途 |
|------|---------|
| `save_memory`（遗留） | 当 `experimental.memoryV2 = false` 时跨会话持久化事实 |
| `get_internal_docs` | 查阅 Gemini CLI 打包的文档 |
| `ask_user` | 向用户提出结构化问题（文本 / 单选 / 多选） |
| `enter_plan_mode` / `exit_plan_mode` | 进入和退出只读 plan mode |
| `update_topic` | 更新当前对话的主题 / 战略意图元数据 |
| `complete_task` | 信号一个 Gemini 子代理已完成并把其结果返回给父代理 |
| `tracker_create_task`、`tracker_update_task`、`tracker_get_task`、`tracker_list_tasks`、`tracker_add_dependency`、`tracker_visualize` | 带依赖和可视化支持的丰富任务跟踪器 |
| `read_mcp_resource`、`list_mcp_resources` | MCP 资源访问 |
