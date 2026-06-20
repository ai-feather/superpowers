# 平台中立配置文件引用 — Phase B 设计

## 背景

Phase A（见 `2026-05-05-platform-neutral-prose-design.md`）已将通用第三人称的 "Claude" 散文替换为代理中立的表述。本阶段处理下一个类别：技能内部对各平台指令文件（CLAUDE.md、AGENTS.md、GEMINI.md）的引用。

插件运行在多个宿主上，每个宿主读取各自的指令文件。凡是技能把 CLAUDE.md 当作唯一文件来命名的地方，都是一种以 Claude Code 为中心的假设，在 Codex / Gemini CLI / OpenCode 上并不成立。

## 范围内

活跃技能中的两行特定内容：

1. **`skills/writing-skills/SKILL.md:58`** — `Project-specific conventions (put in CLAUDE.md)`
2. **`skills/receiving-code-review/SKILL.md:30`** — `"You're absolutely right!" (explicit CLAUDE.md violation)`

## 范围外

- **`skills/using-superpowers/SKILL.md:22, 26`** — 指令优先级列表。该列表已包容地列出三者（CLAUDE.md、GEMINI.md、AGENTS.md），这是正确的：这一节是在对*多平台插件上什么算作用户指令*做出真实断言。无需改动。
- **历史 / 示例制品**：
  - `skills/systematic-debugging/CREATION-LOG.md` — 归属路径（`~/.claude/CLAUDE.md`）是历史事实。
  - `skills/writing-skills/examples/CLAUDE_MD_TESTING.md` — 整个文件是一个测试 CLAUDE.md 内容变体的完整示例。文件名、正文以及来自 `testing-skills-with-subagents.md` 的引用都保留；规范化它们反而破坏示例。
- **平台工具引用** — Phase D 候选：
  - `skills/using-superpowers/SKILL.md:40`（关于 GEMINI.md 的 Gemini CLI 工具映射注记）
  - `skills/using-superpowers/references/gemini-tools.md`（`save_memory` 持久化到 GEMINI.md）

## 替换规则

两个不同的处理，每行一个。

### 规则 1："把项目专属约定放哪里"

`writing-skills/SKILL.md:58`：

- **改前：** `Project-specific conventions (put in CLAUDE.md)`
- **改后：** `Project-specific conventions (put in your instructions file)`

用通用表述而不是固定某一个文件名。不同宿主读取不同文件（CLAUDE.md、AGENTS.md、GEMINI.md 等），技能不应假设某一个。平台工具参考文档（`references/{codex,copilot,gemini}-tools.md`）才是为每个平台点明首选文件的合适位置。

### 规则 2："(explicit CLAUDE.md violation)" 这个括注

`receiving-code-review/SKILL.md:30`：

- **改前：** `"You're absolutely right!" (explicit CLAUDE.md violation)`
- **改后：** `"You're absolutely right!" (explicit instruction-file violation)`

这个括注在发挥真实作用 — 它表明这句话不仅仅是风格糟糕，而是主动违反了许多用户写进指令文件的规则。"Instruction file" 是天然的跨平台术语，统一涵盖 AGENTS.md / CLAUDE.md / GEMINI.md，既保留了原本的信号，又不必挑出某一个文件名，也不至于弱化为 "common"。

## 提交计划

按顺序的原子提交：

1. **`writing-skills/SKILL.md`** — "项目约定放哪里" 那一行中 CLAUDE.md → "your instructions file"
2. **`receiving-code-review/SKILL.md`** — 违规括注中 CLAUDE.md → instruction-file
3. **平台工具参考文档** — 在每个 `references/{codex,copilot,gemini}-tools.md` 中补充各平台首选的指令文件（CLAUDE.md、AGENTS.md、GEMINI.md 等），让读者能把 "your instructions file" 解析到一个真实文件名。

每个提交信息都标明 "Phase B" 和对应切片。

## 验证

每次提交后：

- 阅读所在段落，确认语法与含义仍然通顺。
- `grep -n "CLAUDE\.md" <touched-file>` — 活跃散文中不再有命中（豁免项已记录）。

两次提交都完成后：

- `grep -rn "CLAUDE\.md" skills/` 应只返回已记录的豁免项（CREATION-LOG、CLAUDE_MD_TESTING 及其入向引用、using-superpowers 中的优先级列表）。

## 非目标

- 不要动 `using-superpowers/SKILL.md` 中的优先级列表顺序。重排 CLAUDE.md / GEMINI.md / AGENTS.md 是审美改动，不是替换，不在本范围内。
- 不要重命名 `examples/CLAUDE_MD_TESTING.md` 或修改其内容。
- 不要修改 Gemini CLI 专属的工具引用（Phase D 候选）。

## 实现说明

此处所述的 Phase B 原计划只覆盖三个提交和三个非 Claude Code 平台工具引用。实际实现更进了一步：在提交 `8505703` 中新增了第四个引用 `references/claude-code-tools.md`，以求对称 — 这样 Claude Code 的指令文件约定和工具名列表就与其他平台并列，而不是隐含在周围的技能散文里。这一新增在规格中未曾预料，但与其意图一致。
