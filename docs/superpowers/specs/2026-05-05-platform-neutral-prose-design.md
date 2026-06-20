# 平台中立散文 — Phase A 设计

## 背景

Superpowers 发布到多个代理运行时（Claude Code、Codex、Cursor、OpenCode、Copilot CLI、Gemini CLI）。技能内容和配套文档最初是为 Claude Code 撰写的，在有些地方用 "Claude" 指代任意运行时的代理。OpenAI 的 vendor 化 fork（openai/plugins#217）做过一次全面重写，但其中一些地方实际是错的 — 重写了历史归属路径、模型名称和平台专属安装说明 — 我们要避免这种错误，同时仍然把真正属于附带性的平台中心散文去掉。

整个工作按引用类别分为若干阶段。**本规格只覆盖 Phase A：** 非平台专属语境中提到 "Claude" 的通用第三人称散文。后续阶段（配置文件引用、营销文案、工具名引用）不在本范围内，会有各自的规格。

## 范围内

下列文件中提到 "Claude" 的通用散文：

- `skills/*/SKILL.md` 以及活跃技能目录下的配套 `.md` 文件
- `skills/writing-skills/anthropic-best-practices.md`
- `README.md`（仅在提及是通用散文而非平台营销时）

另加一处造词重命名：**Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**，位于 `skills/writing-skills/SKILL.md`。

## 范围外

- **平台/运行时陈述** — "In Claude Code:"、安装说明、工具映射引用。（Phase D 候选。）
- **配置文件引用** — CLAUDE.md、AGENTS.md、GEMINI.md 优先级列表以及 "把项目约定放哪里" 的提示。（Phase B。）
- **工具名引用** — `Skill`、`Bash`、`Read`、`Task`、`TodoWrite`。技能以 Claude Code 的工具词汇撰写；现有 `references/{codex,copilot,gemini}-tools.md` 文件负责映射。（撰写本规格时，计划是推迟或跳过这些。最终由 Phase E 完成 — 在活跃技能中把工具名替换为动作语言，并围绕同一套词汇统一平台工具引用。）
- **README 中的营销文案** — "Superpowers for Claude Code"、以平台命名的安装章节。（Phase C。）
- **历史制品** — `docs/plans/*.md`、`docs/superpowers/specs/*.md`、`CREATION-LOG.md`。这些都是带日期的即时点文档；重写它们就是改写历史。
- **模型标识符** — Claude Haiku / Sonnet / Opus。这些是真实产品名。
- **文件名 / URL 引用** — `CLAUDE.md`、`claude.com`、`claude-plugin/`、`~/.claude/` 下的路径。
- **`anthropic-best-practices.md` 文件名** — 文件仍按其来源命名，即便我们要改写其中的散文。

## 替换风格

混合使用，使英文读起来自然：

- **第二人称 — "your agent"**，当向技能作者谈论*他们自己的*运行时时
  - "your agent reads the description"
- **第三人称 — "the agent" / "agents" / "an agent"**，当一般性地描述系统行为时
  - "Future agents find your skills"
  - "Use words an agent would search for"
  - "Agents read SKILL.md only when the skill becomes relevant"

选择最适合所在句子的形式；不要为了强制一致而写得别扭。能自然复数化时（"future agents"、"agents read"）就复数化，而不是一律说 "the agent"。

### 保留为 "Claude" 的豁免项

- 模型名：Claude Haiku、Claude Sonnet、Claude Opus
- 文件名和 URL：`CLAUDE.md`、`claude.com`、`~/.claude/`
- 作为运行时本身的品牌平台名 "Claude Code"（在后续阶段处理）

### 造词重命名

- **Claude Search Optimization (CSO) → Skill Discovery Optimization (SDO)**
  - 出现在 `skills/writing-skills/SKILL.md` 中作为章节标题及附近散文。重命名标题、缩写以及任何文件内交叉引用。

## 受影响文件

下表数量基于一次过滤掉豁免项后的 `grep`：

| 文件 | 通用散文提及 |
|------|------------------------|
| `skills/writing-skills/SKILL.md` | ~12（含 CSO 标题与正文） |
| `skills/writing-skills/anthropic-best-practices.md` | ~30 |
| `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` | ~1 — 文件名保留（它是 CLAUDE.md 测试制品）；"Variant C: Claude.AI Emphatic Style" 标题也保留（它是命名某种特定风格的标签） |
| `README.md` | ~1 |

最终清单在实现阶段通过重新跑过滤后的 grep 加以确认。

## 提交计划

按顺序的四个原子提交：

1. **重命名 CSO → SDO**（`skills/writing-skills/SKILL.md`）。机械、孤立，若日后我们对这个术语改变主意也易于回退。
2. **活跃技能散文** — 在 `skills/*/SKILL.md` 及配套 `.md` 中把通用 "Claude" 替换为 "agent" 形式，不含 `anthropic-best-practices.md`。
3. **`anthropic-best-practices.md` 散文** — 同样的替换规则。单独成提交是因为该文件是外部文档的 vendor 化改编；隔离改动便于日后与上游做对账阅读。
4. **README.md 散文** *（仅在过滤后仍残留通用散文提及的情况下）*。没有则跳过。

每个提交信息都标明阶段（"Phase A"）和切片（"rename CSO to SDO"、"agent prose in active skills" 等），使整个系列自解释。

## 验证

每次提交后：

- `grep -rn "Claude" <touched-paths>` — 每条剩余命中都必须落入已记录的豁免项（模型名、文件名、URL、"Claude Code" 平台名、历史制品）。
- 通读被改文件 — 替换不应破坏句子的连贯、代词一致或列表平行结构。
- 无需跑测试；本阶段只改散文。

最终提交后：

- 在真实会话中通览每个改过的技能，确认没有别扭之处。

## 非目标

- 不改行为、结构、标题（CSO→SDO 除外）、示例、代码块或 YAML frontmatter。
- 不新增章节、提示或兼容性说明。
- 编辑时不 "顺手改进" 替换以外的散文。
