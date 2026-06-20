# Codex 工具映射

技能用动作说话（"分派一个子代理"、"创建一个待办"、"读一个文件"）。在 Codex 上，这些解析为下面的工具。

| 技能请求的动作 | Codex 等价物 |
|----------------------|------------------|
| 读文件 | `shell`（例如 `cat`、`head`、`tail`）—— Codex 通过 shell 读文件 |
| 创建 / 编辑 / 删除文件 | `apply_patch`（用于创建、更新、删除的结构化 diff） |
| 运行 shell 命令 | `shell` |
| 搜索文件内容 | `shell`（例如 `grep`、`rg`） |
| 按名查找文件 | `shell`（例如 `find`、`ls`） |
| 抓取 URL | `shell` 配 `curl` / `wget` —— Codex 没有原生 fetch 工具 |
| 搜索网络 | `web_search`（默认启用；可在 `config.toml` 中通过顶层 `web_search` 设置配置——`live`、`cached` 或 `disabled`） |
| 调用技能 | 技能原生加载——直接遵循指令 |
| 分派子代理（`Subagent (general-purpose):` 模板） | `spawn_agent`（见[子代理分派需要多代理支持](#子代理分派需要多代理支持)） |
| 多个并行分派 | 在一个响应里多个 `spawn_agent` 调用 |
| 等待子代理结果 | `wait_agent` |
| 完成后释放子代理槽位 | `close_agent` |
| 任务跟踪（"创建待办"、"标记完成"） | `update_plan` |

## 指令文件

当技能提到"你的指令文件"时，在 Codex 上这是项目根的 **`AGENTS.md`**。Codex 还读 `~/.codex/AGENTS.md` 获取全局上下文，并且 `AGENTS.override.md`（在项目树或 `~/.codex/` 中）存在时优先。Codex 从项目根向下遍历到当前工作目录，拼接沿途找到的 `AGENTS.md` 文件，上限 `project_doc_max_bytes`（默认 32 KiB）。

## 个人技能目录

用户级技能位于 **`$CODEX_HOME/skills/`**（默认 `~/.codex/skills/`）。Codex 还读跨运行时路径 **`~/.agents/skills/`**（与 Copilot CLI 和 Gemini CLI 共享）。当两个目录在同一范围同时存在时，Codex 把它们都作为独立的技能目录加载——Codex 的文档目前没有记录它们之间的优先级。每个技能是一个子目录，包含一个 `SKILL.md`（带 `name` 和 `description` frontmatter）。

## 子代理分派需要多代理支持

加到你的 Codex 配置（`~/.codex/config.toml`）：

```toml
[features]
multi_agent = true
```

这为 `dispatching-parallel-agents` 和 `subagent-driven-development` 等技能启用 `spawn_agent`、`wait_agent` 和 `close_agent`。

遗留说明：`rust-v0.115.0` 之前的 Codex 构建把已派生代理的等待暴露为 `wait`。当前 Codex 用 `wait_agent` 等待已派生代理。`wait` 这个名字现在属于 code-mode 的 `exec/wait`，它按 `cell_id` 恢复一个 yield 的 exec cell；它不是已派生代理的结果工具。

## 环境检测

创建 worktree 或完成分支的技能应在继续前用只读 git 命令检测其环境：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → 已经在一个链接的 worktree 中（跳过创建）
- `BRANCH` 为空 → detached HEAD（无法从沙箱中分支/推送/PR）

见 `using-git-worktrees` 步骤 0 和 `finishing-a-development-branch` 步骤 1，了解每个技能如何使用这些信号。

## Codex App 完成

当沙箱阻止分支/推送操作（外部管理的 worktree 中的 detached HEAD）时，代理提交所有工作并通知用户使用 App 的原生控件：

- **"Create branch"** —— 命名分支，然后通过 App UI 提交/推送/PR
- **"Hand off to local"** —— 把工作转交给用户的本地检出

代理仍可运行测试、暂存文件，并输出建议的分支名、提交信息和 PR 描述供用户复制。
