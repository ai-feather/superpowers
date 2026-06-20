# Claude Code 工具映射

技能用动作说话（"分派一个子代理"、"创建一个待办"、"读一个文件"）。在 Claude Code 上，这些解析为下面的工具。

## 工具

| 技能请求的动作 | Claude Code 工具 |
|----------------------|------------------|
| 读文件 | `Read` |
| 创建新文件 | `Write` |
| 编辑文件 | `Edit` |
| 运行 shell 命令 | `Bash` |
| 搜索文件内容 | `Grep` |
| 按名查找文件 | `Glob` |
| 抓取 URL | `WebFetch` |
| 搜索网络 | `WebSearch` |
| 调用技能 | `Skill` |
| 分派子代理（`Subagent (general-purpose):` 模板） | `Agent`（较旧版本名为 `Task`） |
| 多个并行分派 | 在一个响应里多个 `Agent` 调用 |
| 任务跟踪（"创建待办"、"标记完成"） | `TaskCreate`、`TaskUpdate`、`TaskList`、`TaskGet`；在 `claude -p` / Agent SDK 中用 `TodoWrite`，除非设置了 `CLAUDE_CODE_ENABLE_TASKS=1` |
| 后台进程 / 子代理生命周期（读输出、取消） | `TaskOutput`、`TaskStop` —— 这些与上面的待办工具不同，适用于运行中的 shell、代理和远程会话 |

## 指令文件

当技能提到"你的指令文件"时，在 Claude Code 上这是 **`CLAUDE.md`**。Claude Code 从当前工作目录向上遍历目录树，拼接它沿途找到的每一个 `CLAUDE.md` 和 `CLAUDE.local.md`。标准位置：

| 范围 | 位置 |
|-------|----------|
| 项目（团队共享） | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` |
| 用户全局 | `~/.claude/CLAUDE.md` |
| 本地私有（gitignored） | `./CLAUDE.local.md` |
| 托管策略（组织范围） | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS）、`/etc/claude-code/CLAUDE.md`（Linux/WSL）、`C:\Program Files\ClaudeCode\CLAUDE.md`（Windows） |

CLAUDE.md 文件可以用 `@path/to/file` 导入拉入额外内容（相对或绝对，最多五跳深）。子目录的 `CLAUDE.md` 文件也会被自动发现，并在 Claude Code 读取那些子目录中的文件时按需加载。

Claude Code **不**直接读 `AGENTS.md`。如果一个项目已经为其他代理维护了 `AGENTS.md`，从 `CLAUDE.md` 导入它，这样两个运行时共享同一套指令：

```markdown
@AGENTS.md

## Claude Code

（Claude Code 特定的指令放在这里。）
```

关于路径范围规则和大型项目组织，见 `.claude/rules/`（规则可通过 `paths` frontmatter 限定到特定文件并按需加载）。

## 个人技能目录

用户级技能位于 **`~/.claude/skills/`**。每个技能是一个子目录，包含一个 `SKILL.md`（带 `name` 和 `description` frontmatter）加任何支持文件。Claude Code 目前不识别 Codex、Copilot CLI 和 Gemini CLI 读取的跨运行时路径 `~/.agents/skills/`；如果你未来依赖跨运行时支持，对照 [官方技能文档](https://code.claude.com/docs/en/skills) 验证。
