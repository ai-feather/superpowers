# 测试 Superpowers

Superpowers 有两类截然不同的测试，各自位于独立的目录：

- **`tests/`** —— 插件的非 LLM 代码是否正常工作？针对 brainstorm-server JS、OpenCode 插件加载、codex-plugin 同步以及分析工具的 Bash + node + python 集成测试。
- **`evals/`** —— 代理在真实的 LLM 会话中行为是否正确？由 Python 工具驱动 Claude Code / Codex / Gemini CLI 的真实 tmux 会话，并通过 LLM 执行者和判定者来评估技能的遵循情况。

## 插件测试

位于 `tests/`。目前包括：

- `tests/brainstorm-server/` —— brainstorm 服务端 JS 代码的 node 测试套件。
- `tests/opencode/` —— 针对 OpenCode 插件加载、bootstrap 缓存以及工具注册的 bash 测试。
- `tests/codex-plugin-sync/` —— bash 同步验证。
- `tests/kimi/` —— 针对 Kimi 插件清单接线的 bash/Python 检查。
- `tests/claude-code/test-helpers.sh`、`analyze-token-usage.py` —— 被其余 bash 测试复用的工具。
- `tests/claude-code/test-subagent-driven-development.sh` —— 代理能否描述 SDD 的测试（没有对应的 drill 版本；测试的是描述回忆能力而非行为）。
- `tests/claude-code/test-subagent-driven-development-integration.sh` —— 扩展的 SDD 集成测试，包含 token 分析（drill 覆盖 YAGNI 子集；bash 额外覆盖提交数、Claude Code 任务追踪以及 token 遥测断言）。
- `tests/claude-code/test-worktree-native-preference.sh` —— worktree 技能的 RED-GREEN-REFACTOR 校验（drill 覆盖 PRESSURE 阶段；bash 还覆盖 RED/GREEN 基线）。
- `tests/explicit-skill-requests/` —— Haiku 专属、多轮以及技能名提示触发的测试，覆盖 drill 未覆盖的场景。

通过对应目录的 `run-*.sh` 或 `npm test` 运行插件测试。

## 技能行为评测

位于 `evals/`。drill 是驱动工具；场景位于 `evals/scenarios/*.yaml`。设置方式参见 `evals/README.md`。快速开始：

```bash
cd evals
uv sync --extra dev
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

drill 场景运行较慢（每个 3-30+ 分钟），并且会运行真实的 LLM 会话。它们目前并不属于 CI；自然的后续是采用分层模型（PR 上跑快速子集，夜间及按需跑完整扫描）。
