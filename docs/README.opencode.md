# Superpowers for OpenCode

在 [OpenCode.ai](https://opencode.ai) 中使用 Superpowers 的完整指南。

## 安装

在你的 `opencode.json`（全局或项目级）的 `plugin` 数组中加入 superpowers：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git"]
}
```

重启 OpenCode。插件会通过 OpenCode 的插件管理器安装，并注册全部技能。

通过提问来验证："Tell me about your superpowers"

OpenCode 使用其自己的插件安装机制。如果你同时使用 Claude Code、Codex 或其他宿主，需要分别为每个宿主单独安装 Superpowers。

### 从旧的基于符号链接的安装迁移

如果你之前通过 `git clone` 和符号链接安装过 superpowers，请先移除旧的安装：

```bash
# Remove old symlinks
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# Optionally remove the cloned repo
rm -rf ~/.config/opencode/superpowers

# Remove skills.paths from opencode.json if you added one for superpowers
```

然后按照上面的安装步骤操作。

## 使用

### 查找技能

使用 OpenCode 原生的 `skill` 工具列出全部可用技能：

```
use skill tool to list skills
```

### 加载某个技能

```
use skill tool to load brainstorming
```

### 个人技能

在 `~/.config/opencode/skills/` 下创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### 项目技能

在项目内的 `.opencode/skills/` 下创建项目专属技能。

**技能优先级：** 项目技能 > 个人技能 > Superpowers 技能

## 更新

OpenCode 通过基于 git 的包规格来安装 Superpowers。某些 OpenCode 和 Bun 版本会把解析到的 git 依赖固定在 lockfile 或缓存中，因此重启未必能拿到最新的 Superpowers 提交。如果更新没有生效，请清除 OpenCode 的包缓存，或重新安装该插件。

如需固定到某个具体版本，使用分支或 tag：

```json
{
  "plugin": ["superpowers@git+https://github.com/obra/superpowers.git#v5.0.3"]
}
```

## 工作原理

该插件做两件事：

1. **注入引导上下文**：通过 `experimental.chat.messages.transform` hook，为每次对话注入 superpowers 感知。
2. **注册技能目录**：通过 `config` hook，使 OpenCode 能自动发现全部 superpowers 技能，无需符号链接或手动配置。

### 工具映射

技能以动作的形式表达行为，而不是指名某个具体运行时的工具。在 OpenCode 上，这些动作会解析为：

- "Create a todo" / "mark complete in todo list" → `todowrite`
- `Subagent (general-purpose):` 模板 → OpenCode 的 `task` 工具，并带 `subagent_type: "general"`（代码库探索则用 `"explore"`）
- "Invoke a skill" → OpenCode 原生的 `skill` 工具
- "Read a file" → `read`
- "Create a file" / "edit a file" / "delete a file" → `apply_patch`
- "Run a shell command" → `bash`
- "Search file contents" / "find files by name" → `grep`、`glob`
- "Fetch a URL" → `webfetch`

（已对照所安装的 OpenCode CLI 工具清单进行验证。）

## 故障排查

### 插件未加载

1. 检查 OpenCode 日志：`opencode run --print-logs "hello" 2>&1 | grep -i superpowers`
2. 核对你的 `opencode.json` 中的插件配置行是否正确
3. 确保你运行的是较新版本的 OpenCode

### Windows 安装问题

某些 Windows 版本的 OpenCode 在基于 git 的插件规格上存在上游安装器问题，包括 `git+https` URL 的缓存路径问题，以及 Bun 在普通终端中能找到 `git.exe`、却在此处找不到的问题。如果 OpenCode 无法安装该插件，可以尝试用系统 npm 安装，并让 OpenCode 指向本地包：

```powershell
npm install superpowers@git+https://github.com/obra/superpowers.git --prefix "$HOME\.config\opencode"
```

然后在 `opencode.json` 中使用已安装包的路径：

```json
{
  "plugin": ["~/.config/opencode/node_modules/superpowers"]
}
```

### 找不到技能

1. 使用 OpenCode 的 `skill` 工具列出可用技能
2. 检查插件是否在加载（见上文）
3. 每个技能都需要一个带有合法 YAML frontmatter 的 `SKILL.md` 文件

### 引导上下文未出现

1. 检查 OpenCode 版本是否支持 `experimental.chat.messages.transform` hook
2. 修改配置后重启 OpenCode

## 获取帮助

- 提交 issue：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode 文档：https://opencode.ai/docs/
