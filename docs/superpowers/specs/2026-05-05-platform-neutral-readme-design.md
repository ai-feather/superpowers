# 平台中立 README 排序 — Phase C 设计

## 背景

Phase A 和 Phase B（见 `2026-05-05-platform-neutral-prose-design.md` 与 `2026-05-05-platform-neutral-config-refs-design.md`）已将 README 中通用的 Claude 散文和配置文件引用做了中性化处理。剩余的平台倾向信号在版式上：README 的两处平台列表把 Claude Code 放在首位，其余位置也不是严格按字母顺序排列。

本阶段修正排序。不改动散文。

## 范围内

1. **快速开始平台列表**（`README.md:7`）— 支持宿主的内联链接列表
2. **安装章节排序**（`README.md:35–152`）— 各宿主的安装子章节

## 范围外

- 散文、市场名称、插件 ID、URL — 现状在事实上都正确。
- Claude Code 章节的视觉权重（它有两个子章节 — 官方 Anthropic marketplace 与 Superpowers marketplace）。两者都是真实安装路径；合并它们会掩盖准确信息。
- 各安装块内部的章节标题和内容 — 只改变块的顺序。

## 替换

两处列表都重新排序为严格字母顺序：

| 旧顺序 | 新顺序 |
|-----------|-----------|
| Claude Code | Claude Code |
| Codex CLI | Codex App |
| Codex App | Codex CLI |
| Factory Droid | Cursor |
| Gemini CLI | Factory Droid |
| OpenCode | Gemini CLI |
| Cursor | GitHub Copilot CLI |
| GitHub Copilot CLI | OpenCode |

三处移动：Codex App 与 Codex CLI 互换；Cursor 上移两位；GitHub Copilot CLI 上移一位。

Claude Code 仍居首位，纯属字母巧合（`Cl…` 在 `Co…` 之前）。

## 提交计划

一个原子提交覆盖两处列表，因为只改一处会让快速开始和安装章节之间不一致。

## 验证

- 快速开始的锚点（`#claude-code`、`#codex-app` 等）仍然指向现有的 `### …` 标题 — 未重命名任何标题。
- 各安装子章节的正文在前后字节完全一致；只改变了位置。
- `git diff README.md` 只显示章节移动，没有内容编辑。
