# Copilot CLI 工具映射

技能用动作说话（"分派一个子代理"、"创建一个待办"、"读一个文件"）。在 Copilot CLI 上，这些解析为下面的工具。

| 技能请求的动作 | Copilot CLI 等价物 |
|----------------------|----------------------|
| 读文件 | `view` |
| 创建 / 编辑 / 删除文件 | `apply_patch`（Copilot CLI 没有单独的创建/编辑/写入工具） |
| 运行 shell 命令 | `bash` |
| 搜索文件内容 | `rg`（ripgrep；Copilot CLI 不暴露 `grep` 工具） |
| 按名查找文件 | `glob` |
| 抓取 URL | `web_fetch` |
| 搜索网络 | `web_search` |
| 调用技能 | `skill` |
| 分派子代理（`Subagent (general-purpose):` 模板） | `task` 配 `agent_type: "general-purpose"`（其他可接受类型：`explore`、`task`、`code-review`、`research`、`configure-copilot`） |
| 多个并行分派 | 在一个响应里多个 `task` 调用 |
| 子代理状态/输出/控制 | `read_agent`、`list_agents`、`write_agent` |
| 任务跟踪（"创建待办"、"标记完成"） | `update_todo` |
| 进入 / 退出 plan mode | 无等价物——留在主会话中 |

## 指令文件

当技能提到"你的指令文件"时，在 Copilot CLI 上这是仓库根的 **`AGENTS.md`**。如果 `AGENTS.md` 和 `.github/copilot-instructions.md` 同时存在，Copilot 两者都读。

## 个人技能目录

用户级技能位于 **`~/.copilot/skills/`**。Copilot CLI 还识别跨运行时别名 **`~/.agents/skills/`**，与 Codex 和 Gemini CLI 共享。每个技能是一个子目录，包含一个 `SKILL.md`（带 `name` 和 `description` frontmatter）。

## 异步 shell 会话

Copilot CLI 支持持久的异步 shell 会话：

| 工具 | 用途 |
|------|---------|
| `bash` 配 `mode: "async"`（可选 `detach: true`） | 在后台启动长时间运行的命令；返回一个 `shellId` |
| `write_bash` | 向运行中的异步会话发送输入 |
| `read_bash` | 从异步会话读输出 |
| `stop_bash` | 终止异步会话 |
| `list_bash` | 列出所有活跃的 shell 会话 |

## 其他 Copilot CLI 工具

| 工具 | 用途 |
|------|---------|
| `store_memory` | 为未来会话持久化关于代码库的事实 |
| `report_intent` | 用当前意图更新 UI 状态行 |
| `sql` | 查询会话的 SQLite 数据库（待办、元数据） |
| `fetch_copilot_cli_documentation` | 查阅 Copilot CLI 文档 |
| GitHub MCP 工具（`github-mcp-server-*`） | 原生 GitHub API 访问（issue、PR、代码搜索） |
