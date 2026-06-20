# Pi 工具映射

技能用动作说话（"分派一个子代理"、"创建一个待办"、"读一个文件"）。在 Pi 上，这些解析为下面的工具。

| 技能请求的动作 | Pi 等价物 |
| --- | --- |
| 调用技能 | Pi 原生技能：用 `read` 加载相关 `SKILL.md`，或让人用 `/skill:name` |
| 读文件 | `read` |
| 创建文件 | `write` |
| 编辑文件 | `edit` |
| 运行 shell 命令 | `bash` |
| 搜索文件内容 | `grep`（可用时）；否则 `bash` 配 `rg`/`grep` |
| 按名查找文件 | `find` 或 `bash` 配 shell glob |
| 列出文件和子目录 | `ls`（可用时）；否则 `bash` 配 `ls` |
| 分派子代理（`Subagent (general-purpose):` 模板） | 用一个已安装的子代理工具，例如 `pi-subagents` 的 `subagent`（如果可用） |
| 任务跟踪（"创建待办"、"标记完成"） | 用一个已安装的待办/任务工具（如果可用），否则在计划或 `TODO.md` 中跟踪任务 |

## 技能

Pi 从配置的技能目录和已安装的 Pi 包发现技能。一个 Superpowers Pi 包应通过其 `pi.skills` manifest 条目暴露 `skills/`。Pi 不暴露 Claude Code 的 `Skill` 工具，但代理仍应遵循 Superpowers 规则：当技能适用时，在响应前加载并遵循它。

## 子代理

Pi 核心不内置标准子代理工具。`pi-subagents` 包是一个强力的可选伴侣，提供 `subagent` 工具，带单代理、链式、并行、异步、forked-context 和 resume/status 工作流。如果没有子代理工具可用，不要捏造 `Task` 调用；在当前会话中顺序执行，或说明可选的子代理能力未安装。

## 任务列表

Pi 核心不内置标准任务列表工具。如果安装了待办/任务扩展，用其文档化的工具。否则用 Superpowers 计划文件、Markdown 清单，或仓库本地的 `TODO.md` 做任务跟踪。较旧的 Superpowers 文档可能提到 `TodoWrite`；把它当作上面的任务跟踪动作。
