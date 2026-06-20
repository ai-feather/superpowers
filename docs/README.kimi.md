# Superpowers for Kimi Code

在 [Kimi Code](https://github.com/MoonshotAI/kimi-code) 中使用 Superpowers 的完整指南。

## 安装

Superpowers 可在 Kimi Code 的插件市场中获取。

打开插件管理器：

```text
/plugins
```

进入 `Marketplace` > `Superpowers` 并安装。

你也可以从本仓库直接安装：

```text
/plugins install https://github.com/obra/superpowers
```

如需对尚未发布的 `dev` 分支进行验证，请显式指定分支：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

Kimi Code 会把插件改动应用到新会话中。在安装、更新、启用、禁用或重载插件之后，请用 `/new` 启动一个全新会话。

## 工作原理

Kimi 插件清单位于 `.kimi-plugin/plugin.json`。

该清单做三件事：

1. 将 Kimi Code 指向既有的 `skills/` 目录。
2. 通过 `sessionStart.skill` 在会话启动时加载 `using-superpowers`。
3. 通过 `skillInstructions` 提供 Kimi 专属的工具映射。

Kimi Code 直接从本仓库读取 Superpowers 的技能。这里没有被复制的技能、符号链接、hooks 或额外的运行时依赖。

## 工具映射

技能以动作的形式描述行为，而不是硬编码某个运行时的工具名。在 Kimi Code 上，这些动作会解析为：

- "Ask the user" / "ask clarifying questions" -> `AskUserQuestion`
- "Create a todo" / "mark complete in todo list" -> `TodoList`
- "Dispatch a subagent" -> `Agent`
- "Invoke a skill" -> Kimi Code 原生的 `Skill` 工具
- "Read a file" / "write a file" / "edit a file" -> `Read`、`Write`、`Edit`
- "Run a shell command" -> `Bash`
- "Search file contents" -> `Grep`
- "Find files by path or pattern" -> `Glob`
- "Fetch a URL" -> `FetchURL`
- "Search the web" -> `WebSearch`

## 更新

使用 Kimi Code 的插件管理器：

```text
/plugins
```

选择 Superpowers 并从中更新。更新后请用 `/new` 启动一个全新会话。

## 故障排查

### 插件未加载

1. 运行 `/plugins info superpowers` 并查看诊断信息。
2. 确认插件已启用。
3. 在安装或更新之后用 `/new` 启动一个全新会话。

### 直接 GitHub 安装使用了旧版本

当存在 GitHub release 时，Kimi Code 会为裸仓库 URL 安装最新的 GitHub release。要在下一次 Superpowers 发布之前测试未发布的改动，请显式指定分支安装：

```text
/plugins install https://github.com/obra/superpowers/tree/dev
```

### 技能未触发

1. 确认 `/plugins info superpowers` 显示插件已启用。
2. 用 `/new` 启动一个全新会话。
3. 尝试验收提示词：`Let's make a react todo list`。一个可用的安装应当在写代码之前加载 `brainstorming`。
