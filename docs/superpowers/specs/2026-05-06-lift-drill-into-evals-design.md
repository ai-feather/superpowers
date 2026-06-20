# 将 drill 提升到 superpowers 中作为 `evals/` —— 设计

## 背景

Drill 是一个 Python 技能合规基准，位于其独立仓库 `obra/drill`。它驱动真实的 tmux 会话，以 LLM actor 模拟你的搭档，对生成的 transcript 运行 LLM verifier，并按场景报告 pass/fail。它支持 Claude Code、Codex、Gemini CLI，以及（按最近的提交）OpenCode 和 Copilot CLI。

Drill 已经是 superpowers 的事实评估宿主。drill 仓库中的 PRI-1397 提交序列把约 22 个 superpowers bash 测试提升为 drill 场景，而最近一次 superpowers 提交（`a2292c5`）显式移除了一个冗余 bash 测试，消息是 *"replaced by drill behavioral coverage"*。迁移势能已经存在；本规格完成它。

本工作把 drill 移入 superpowers 的 `evals/` 下，在逐文件核验 drill 场景覆盖之后删除冗余 bash 测试，并更新文档，使贡献者落在新的结构上。

## 目标

1. `evals/` 是 superpowers 中规范的评估宿主 —— 完整的 drill 源码、场景、fixtures、prompts、backend 配置和测试。
2. `superpowers/tests/` 中已逐项验证为被 drill 场景 100% 覆盖的 bash 测试将被删除；其余保留。
3. `tests/`（插件基础设施：bash + node + python 集成测试）与 `evals/`（带 actor + verifier 的 LLM 行为）之间的分工是有意义且有文档的。
4. 顶层文档（`README.md`、`CLAUDE.md`、`docs/testing.md`）把贡献者指向正确位置。
5. 独立的 `obra/drill` 仓库继续存在（本 PR 不动它），并在此 PR 合并后作为单独的手动步骤归档。

## 非目标

- **CI 集成。** 此处仅手动运行。自然的后续是"分层"方案：每个 PR 上跑快速子集，夜间 + 按需跑全量。这需要 API 预算决策、GitHub Actions secrets，以及一个装有 `tmux` + `node` + `python` + `claude` / `codex` / `gemini` CLIs 的 runner 镜像。范围之外。
- **场景与技能共置。** 场景继续集中在 `evals/scenarios/`。如果之后决定每个技能应拥有自己的场景，那是一次 path-find-and-replace 操作；YAML 格式不变。
- **重命名内部 Python 包**（`drill` → `evals`）。目录是 `evals/`（面向用户）；Python 包保留其 `drill` 名字以保持 diff 小。`evals/README.md` 里有一句简短说明。
- **Drill 仓库归档。** 本 PR 不动 `obra/drill`。合并后，drill 仓库会被手动归档（GitHub 上只读，README 指向 `obra/superpowers/evals/`）。
- **将 `tests/claude-code/analyze-token-usage.py` 提升到 `evals/bin/`。** 有用的工具，不是测试代码。可以之后移；本 PR 不要求。

## 分支

从 `dev` 切出 `f/evals-lift`。本工作与开着的 `f/cross-platform` PR 相互独立 —— 除可能的 `README.md` 外没有共享文件改动，而 `README.md` 足够小，冲突可以在合并时解决。

## 迁移后的架构

```
superpowers/
  evals/                              ← 新增（完整 drill 副本）
    pyproject.toml                    (Python 3.11，uv 管理)
    uv.lock
    .gitignore                        (drill 自己的；results/, .venv/, .env)
    README.md                         (原 drill 的 README；安装说明已更新)
    CLAUDE.md                         (原 drill 的 CLAUDE.md；路径已更新)
    docs/
      design.md                       (drill 的设计 —— 原样保留，与本规格交叉引用)
      manual-testing.md
      pressure-and-red-testing.md
    drill/                            (Python 包；名字保留；cli, engine, actor, verifier 等)
    backends/                         (claude-*.yaml, codex.yaml, gemini.yaml)
    scenarios/                        (32+ 个 YAML 场景)
    setup_helpers/                    (15 个 Python 辅助；create_base_repo, sdd_*, spec_*, worktree 等)
    fixtures/                         (template-repo, sdd-go-fractals, sdd-svelte-todo)
    prompts/                          (actor.md, verifier.md)
    bin/                              (断言辅助脚本：tool-called, tool-count 等)
    tests/                            (drill 自己的 pytest 套件)

  tests/                              ← bash 测试默认保留
    brainstorm-server/                ← 保留（brainstorm-server JS 代码的 node 测试）
    opencode/                         ← 保留（插件加载测试）
    codex-plugin-sync/                ← 保留（同步校验）
    claude-code/                      ← 多数保留 —— 见删除门
    explicit-skill-requests/          ← 保留，除非已验证被替代
    skill-triggering/                 ← 保留，除非已验证被替代
    subagent-driven-dev/              ← 保留，除非已验证被替代

  docs/
    testing.md                        ← 已更新（拆分为 "Plugin tests" + "Skill behavior evals"）
    superpowers/
      specs/
        2026-05-06-lift-drill-into-evals-design.md   ← 本规格

  README.md                           ← Contributing 节加一条指向 evals/ 的小提示
  CLAUDE.md                           ← 一行 "Eval harness lives at evals/" 指针
```

本 PR 之后，`tests/` 与 `evals/` 目录承担明显不同的角色：

- **`tests/`** —— 插件的非 LLM 代码能否正常工作？brainstorm-server JS 代码、OpenCode 插件加载、codex-plugin-sync 同步校验的单元与集成测试。Bash + node + python。
- **`evals/`** —— 代理在真实 LLM 会话上的行为是否正确？带 actor + verifier 的 Drill 场景。仅 Python，运行真实 tmux 会话。

## 删除门（针对每个 bash 测试）

仅当某个 drill 场景可验证地覆盖了某 bash 测试的全部断言时，才删除该 bash 测试。实现计划按文件记录此项验证：读 bash 测试，列出其检查项，找到对应 drill 场景，确认每个检查项都有对应的 `verify.assertions` 或 `verify.criteria` 条目。只要有任一检查项缺失，选项就是要么扩展 drill 场景，要么保留该 bash 测试。默认保留。

**初步覆盖映射**（基于提交消息；删除前需要逐文件验证）：

| Bash 测试 | 声称的 drill 替代 | 覆盖状态 |
|-----------|---------------------------|-----------------|
| `tests/skill-triggering/prompts/*`（6 个 prompt 文件） | `triggering-*.yaml`（6 个场景） | 候选 —— 删除前逐 prompt 验证 |
| `tests/skill-triggering/run-test.sh`, `run-all.sh` | 不适用（是运行器，不是测试） | **保留** —— runner 脚本 |
| `tests/explicit-skill-requests/prompts/please-use-brainstorming.txt` | 需要验证 —— drill 尚无明显的对应项 | 大概率 **保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/prompts/use-systematic-debugging.txt` | 需要验证 —— drill 尚无明显的对应项 | 大概率 **保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/run-claude-describes-sdd.sh` | 部分对应 → `mid-conversation-skill-invocation.yaml` | 候选 —— 逐脚本验证 |
| `tests/explicit-skill-requests/run-haiku-test.sh` | 无 drill 场景覆盖 Haiku 专有行为 | **保留** |
| `tests/explicit-skill-requests/run-multiturn-test.sh`, `run-extended-multiturn-test.sh` | 无 drill 场景覆盖多轮累积 | **保留**，除非添加 drill 场景 |
| `tests/explicit-skill-requests/run-test.sh`, `run-all.sh` | 不适用（运行器） | **保留** |
| `tests/subagent-driven-dev/go-fractals/`, `tests/subagent-driven-dev/svelte-todo/` | `sdd-go-fractals.yaml`, `sdd-svelte-todo.yaml` | 候选 —— 删除前验证（这些包含关于测试套件通过的真实断言） |
| `tests/claude-code/test-document-review-system.sh` | `spec-reviewer-catches-planted-flaws.yaml` | 候选 —— 删除前验证 |
| `tests/claude-code/test-requesting-code-review.sh` | `code-review-catches-planted-bugs.yaml` | 候选 —— 删除前验证 |
| `tests/claude-code/test-subagent-driven-development-integration.sh` | `sdd-rejects-extra-features.yaml`（YAGNI 子集） | **部分** —— bash 测试还断言 ≥3 次提交 / `npm test` 通过 / 运行 `analyze-token-usage.py`。drill 场景断言 forbidden-exports + reviewer-as-gate。大体不相交 —— 几乎肯定 **保留 + 扩展 drill 场景**。 |
| `tests/claude-code/test-subagent-driven-development.sh` | 元/文档测试（让代理*描述* SDD）；无 drill 场景覆盖描述类测试 | **保留**，除非添加 drill 场景 |
| `tests/claude-code/test-worktree-native-preference.sh` | `worktree-creation-under-pressure.yaml` | 候选 —— 删除前验证 |
| `tests/claude-code/test-helpers.sh`, `run-skill-tests.sh`, `analyze-token-usage.py` | 不适用（工具，不是测试） | **保留** —— 库/工具 |

## 验证协议（子代理把关）

实现计划中的每项变更在提交前都由一个独立的子代理交叉核对。

| 变更类别 | 子代理验证 |
|----------------|----------------------|
| 每次删除 bash 测试 | 派发子代理，输入：(a) bash 测试文件内容，(b) 候选 drill 场景 YAML，(c) prompt：*"List every assertion the bash test makes. List every verify entry in the drill scenario. For each bash assertion, find a matching drill check or report it as unmatched. Output a per-assertion table."* 子代理的输出就是门控 —— 只有每条 bash 断言都有匹配时才删除。 |
| 初始 `evals/` 拷贝 | 子代理验证：(a) 拷贝的 drill SHA 已记录在 lift 提交消息中以可审计来源；(b) **逐文件 SHA-256 校验和**对每个文件与 drill 仓库一致（不只是文件数量）；(c) 排除路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、任何 `.private-journal/`）不存在于 `evals/`；(d) 所有 backend YAML 引用的路径在移动后仍存在；(e) `pyproject.toml`、`uv.lock`、`.gitignore` 完好。 |
| Drill 自己的 pytest 套件 | 子代理在路径默认值变更后运行 `cd evals && uv run pytest`。drill 自带 pytest 套件于 `evals/tests/`，包含 `test_backend.py`，它覆盖 `SUPERPOWERS_ROOT` 环境变量行为 —— 这些测试必须更新以匹配辅助函数并继续通过。 |
| 删除后的引用清理 | 子代理在整个 superpowers 树中（排除 `node_modules/`、`.venv/` 和 `evals/`）grep 被删除的 bash 测试路径的引用。搜索目标：`docs/`、`docs/superpowers/plans/`、`RELEASE-NOTES.md`、`CLAUDE.md`、`GEMINI.md`、`AGENTS.md`、`README.md`、`.github/`、`scripts/`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`lefthook.yml`。任何命中要么更新，要么暴露一个被漏掉的依赖。 |
| 路径默认值变更（`SUPERPOWERS_ROOT` 默认值） | 子代理在路径变更后至少运行一个便宜的 drill 场景（例如 `triggering-test-driven-development`）并确认它仍然通过。真实验证，而不只是代码审查。 |
| PR 前最终对抗式审查 | 两个子代理并行，"找出最多合理问题者得 5 分"框架 —— 与跨平台 PR 使用的相同协议。源码和行为都要验证。 |

每个子代理任务在实现计划里都有自己的一条要点，含明确的输入和通过标准。子代理的输出被摘录到相关提交消息（"Subagent verification: …"）中，使链条可审计。

## 具体的路径/配置编辑

**已在编写本规格前验证。** `drill/cli.py` 定义 `PROJECT_ROOT = Path(__file__).parent.parent`。移动后，`cli.py` 位于 `evals/drill/cli.py`，因此 `PROJECT_ROOT` 解析为 `evals/`，而 `PROJECT_ROOT.parent` 解析为 superpowers 仓库根。这就是 `SUPERPOWERS_ROOT` 应当默认取的值。

**YAML 替换审计。** 只有四个 `claude*.yaml` backend 配置把 `${SUPERPOWERS_ROOT}` 插值到 `args`（用于 `--plugin-dir` flag）；`codex.yaml` 和 `gemini.yaml` 只在 `required_env` 里列出 `SUPERPOWERS_ROOT`（由 pre/post-run 钩子中的 `engine.py:233` / `setup.py:25` 的 `os.environ["SUPERPOWERS_ROOT"]` 查找消费）。辅助函数对 `os.environ` 的修改覆盖了两条代码路径。

| 文件 | 当前 | 之后 |
|------|---------|-------|
| `drill/cli.py` | 模块导入处 `load_dotenv(PROJECT_ROOT / ".env")`；没有关于 `SUPERPOWERS_ROOT` 的内容 | `load_dotenv` 之后，调用新辅助 `_set_superpowers_root_default()`，当且仅当尚未设置时把 `os.environ["SUPERPOWERS_ROOT"]` 设为 `str(PROJECT_ROOT.parent)`。顺序：`load_dotenv` → 设置默认 → click group 定义。 |
| `drill/engine.py:233`, `drill/setup.py:25` | 直接访问 `os.environ["SUPERPOWERS_ROOT"]`（未设置时 KeyError） | 不变。CLI 启动钩子保证环境变量在 engine/setup 执行时已被设置。 |
| `backends/claude*.yaml`（5 个文件） | `${SUPERPOWERS_ROOT}` 替换进 `args`，用于 `--plugin-dir` | 不变。YAML 替换在 backend 加载时读 `os.environ`，这发生在 CLI 启动之后。 |
| `backends/codex.yaml`, `backends/gemini.yaml` | `SUPERPOWERS_ROOT` 仅在 `required_env` 中 | 从 `required_env` 中移除（辅助会提供它）。`claude*.yaml` 为向后兼容保留 `required_env`（环境变量仍可作为覆盖）。 |
| `evals/tests/test_backend.py` | 测试断言 `SUPERPOWERS_ROOT` 在 `required_env` 列表中，外加路径解析测试 | 更新测试以匹配新契约：辅助提供默认值，环境变量覆盖仍然有效，codex/gemini 不再需要 `required_env`。 |
| `evals/README.md` | "export SUPERPOWERS_ROOT=/path/to/superpowers" | 去掉这行 export；说明环境变量自动默认为 `evals/` 的父目录；指出唯一必需的设置是 `ANTHROPIC_API_KEY`（或 `OPENAI_API_KEY` / Gemini 鉴权）。 |
| `evals/CLAUDE.md` | 同上 | 同上 |
| `evals/.gitignore` | drill 现有的模式（`results/`、`.venv/`、`__pycache__/`、`.env`、`*.pyc`、`*.egg-info/`、`dist/`、`build/`、`.claude/`） | 原样拷贝。模式相对于文件位置，因此在 `evals/` 下也正确生效。 |
| `evals/lefthook.yml` | drill 自带 `lefthook.yml`，定义 `pre-commit: uv run ruff check && uv run ty check` | 移到 `evals/lefthook.yml`。要么 (a) 在 superpowers 根安装 lefthook 并让它 federate 到 `evals/lefthook.yml`，要么 (b) 文档化贡献者手动运行 `cd evals && lefthook run pre-commit`。**实现中的决定：为简单起见选 (b)** —— superpowers 的顶层工作流不变。 |

`.env` 位置：保留 `evals/.env`（被 gitignore）。贡献者从那里 source，或在 shell 环境中设置 `ANTHROPIC_API_KEY`。

**需要小幅增补的 superpowers 顶层文件：**

- `superpowers/.gitignore`：加 `evals/results/`、`evals/.venv/`、`evals/.env`（双保险；evals/.gitignore 本地已覆盖这些）。
- `superpowers/CLAUDE.md`：加一行指针 "Eval harness lives at `evals/` — see `evals/README.md`"，让代理能发现它。
- `superpowers/docs/testing.md`：拆分为 "## Plugin tests"（现有 tests/ 内容，裁剪已删除测试的引用）和 "## Skill behavior evals"（一段摘要 + 指向 `evals/` 的指针）。
- `superpowers/README.md`：在 Contributing 节加一行，指向 `evals/` 做技能行为测试。

## 迁移顺序

每一步是一个独立提交（或一小组提交）。第 2 步是最大的单个提交（原样复制 drill）；后续步骤都小而原子。

```
1. 从 `dev` 切出 (f/evals-lift)

2. 把 drill 仓库拷贝进 evals/（单个提交，便于回滚）
   ├─ 在拷贝时记录 drill SHA → 写进提交消息
   ├─ 使用 `rsync -a --exclude=.git --exclude=.venv --exclude=results
   │  --exclude=.env --exclude=__pycache__ --exclude='*.egg-info'
   │  --exclude=.private-journal /path/to/drill/ evals/`
   │  （选 rsync 而非 `cp -r`，以显式排除；用
   │  `find evals -name '.git' -type d` 验证返回空）
   ├─ 子代理门控：每个未排除文件的 SHA-256 校验和与 drill 仓库
   │  一致；排除路径不存在于 evals/
   └─ 冒烟检查：`cd evals && uv sync` 成功（只证明能装；
      不是行为测试）

3. 更新路径默认值
   ├─ 向 drill/cli.py 添加 _set_superpowers_root_default() 辅助
   ├─ 在 load_dotenv 之后、click group 定义之前接入
   ├─ 更新 evals/README.md 和 evals/CLAUDE.md（去掉 SUPERPOWERS_ROOT 安装步骤）
   ├─ 从 codex.yaml/gemini.yaml 的 required_env 中移除 SUPERPOWERS_ROOT
   │  （在 claude*.yaml 中保留作为覆盖）
   └─ 更新 evals/tests/test_backend.py 以匹配新契约

4. 从新位置验证（两项检查）
   ├─ 跑 drill 自己的 pytest：`cd evals && uv run pytest` —— 必须通过
   └─ 跑一个便宜的 drill 场景：`cd evals && uv run drill run
      triggering-test-driven-development -b claude` —— 必须通过。
      真实行为验证，不只是代码审查。

5. Bash 测试删除阶段 —— 逐文件加子代理门控
   对候选删除清单中的每个文件：
   a. 子代理对比 bash 测试断言与 drill 场景 verify 块
   b. 通过条件：每条 bash 断言都有匹配的 drill 检查
   c. 若通过 → 删除该 bash 测试文件（每文件或每一致组一个提交）
   d. 若失败 → 要么扩展 drill 场景（独立提交 + 验证），要么
      保留该 bash 测试（不提交）

6. 过时引用清理
   ├─ 子代理在 superpowers 树（排除 node_modules/, .venv/,
   │  evals/）中 grep 被删除的文件路径
   ├─ 搜索目标：docs/, docs/superpowers/plans/, RELEASE-NOTES.md,
   │  CLAUDE.md, GEMINI.md, AGENTS.md, README.md, .github/, scripts/,
   │  .opencode/INSTALL.md, .codex-plugin/INSTALL.md, lefthook.yml
   ├─ 更新活动引用（如 docs/testing.md、README.md 安装）
   └─ docs/superpowers/plans/*.md 和 RELEASE-NOTES.md 中的历史引用
      被保留并加简短标注（"(test removed; behavior covered by drill
      scenario X)"），而不是重写 —— 这些是有日期的产物，不是活文档。

7. 顶层文档
   ├─ docs/testing.md 拆分
   ├─ CLAUDE.md 指针
   └─ README.md Contributing 节

8. 重跑冒烟检查（回归门控）
   ├─ `cd evals && uv run pytest`
   └─ `cd evals && uv run drill run triggering-test-driven-development -b claude`

9. 最终对抗式审查
   └─ 两个并行子代理，完整 diff，"找出最多合理问题者得 5 分"框架。
      推送前处理发现的问题。

10. 推送分支 + 向 dev 开 PR
    └─ PR 描述包含：拷贝时锁定的 drill SHA、归档行动项
       （"合并后：归档 obra/drill，在 obra/superpowers/evals/
       加 README 指针"）、逐删除文件覆盖凭据。
```

## 验证（实现后）

实现计划必须展示：

- 第 2 步后，所有未排除的 drill 源文件都存在于 `evals/`（子代理 **逐文件 SHA-256 校验和 diff**，对比 `obra/drill@<recorded-sha>`）。
- 排除路径（`.git/`、`.venv/`、`results/`、`.env`、`__pycache__/`、`*.egg-info/`、`.private-journal/`）不存在于 `evals/`。
- 第 2 步的提交消息记录了 drill 源 SHA。
- 在未设置 `SUPERPOWERS_ROOT` 的情况下 `cd evals && uv sync` 成功。
- `cd evals && uv run pytest` 通过（drill 自己的 pytest 套件）。
- `cd evals && uv run drill list` 返回与独立 drill 仓库在记录 SHA 上相同的场景计数。
- `cd evals && uv run drill run triggering-test-driven-development -b claude` 通过（证明路径默认值端到端可用）。
- 对每个被删除的 bash 测试：提交消息中含子代理验证表，展示每条断言如何映射到一条 drill 检查。
- 在活 superpowers 文档中 grep 被删除文件路径返回零命中（第 6 步之后）；`docs/superpowers/plans/*.md` 和 `RELEASE-NOTES.md` 中的历史引用被标注，而非重写。
- `docs/testing.md` 同时含 "Plugin tests" 和 "Skill behavior evals" 两节。
- drill 仓库的历史未动；`obra/drill` 不受本 PR 影响。
- PR 描述点名合并后归档 `obra/drill` 的行动项。

## 待澄清问题

无。所有澄清性决定都已做出：

| 问题 | 决定 |
|----------|----------|
| drill 放在 superpowers 的哪里？ | `evals/`（从 drill 改名）；独立仓库作为单独步骤归档 |
| 冗余 bash 测试的命运？ | 逐文件删除并附子代理覆盖验证；默认保留 |
| 场景布局？ | 集中于 `evals/scenarios/` |
| Python 工具链放置？ | 自包含于 `evals/` |
| CI 集成？ | 本 PR 仅手动；记录未来路径 |
| 迁移机制？ | 纯拷贝；drill 仓库历史保留于归档仓库，不在仓库内 |
| 内部 Python 包名？ | 保留为 `drill`（目录是 `evals/`） |
| 分支策略？ | 从 `dev` 独立切出（不堆叠在 `f/cross-platform` 上） |
