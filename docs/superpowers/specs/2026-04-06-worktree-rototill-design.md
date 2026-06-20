# Worktree Rototill：检测并让位

**日期：** 2026-04-06
**状态：** 草稿
**工单：** PRI-974
**包含：** PRI-823（Codex App 兼容性）

## 问题

Superpowers 对 worktree 管理有自己的主张 —— 特定路径（`.worktrees/<branch>`）、特定命令（`git worktree add`）、特定清理（`git worktree remove`）。与此同时，Claude Code、Codex App、Gemini CLI 和 Cursor 都提供原生 worktree 支持，各有各的路径、生命周期管理与清理。

这带来三种失败模式：

1. **重复** —— 在 Claude Code 上，技能做了 `EnterWorktree`/`ExitWorktree` 已经做的事
2. **冲突** —— 在 Codex App 上，技能试图在已被管理的 worktree 之内再创建 worktree
3. **幻影状态** —— 技能创建的 `.worktrees/` worktree 对宿主不可见；宿主创建的 `.claude/worktrees/` worktree 对技能不可见

对于没有原生支持的宿主（Codex CLI、OpenCode、Copilot standalone），superpowers 填补了一个真实的空白。这个技能不应消失 —— 它应当在存在原生支持时让位。

## 目标

1. 当存在原生宿主 worktree 系统时让位给它们
2. 继续为缺乏原生支持的宿主提供 worktree 支持
3. 修复 finishing-a-development-branch 的三个已知 bug（#940、#999、#238）
4. 让 worktree 创建变为可选而非强制（#991）
5. 用平台中立的语言替换硬编码的 `CLAUDE.md` 引用（#1049）

## 非目标

- 每个 worktree 的环境约定（`.worktree-env.sh`、端口偏移） —— Phase 4
- 用于路径强制的 PreToolUse 钩子 —— Phase 4
- 多仓库 worktree 文档 —— Phase 4
- 针对	worktree 的 brainstorming 清单变更 —— Phase 4
- `.superpowers-session.json` 元数据追踪（PR #997 那个有趣的想法，v1 不需要）
- 钩子符号链接进 worktree（PR #965 的想法，独立关注点）

## 设计原则

### 检测状态，而非平台

用 `GIT_DIR != GIT_COMMON` 判定"我是否已在一个 worktree 里？"，而不是靠嗅探环境变量来识别宿主。这是一个稳定的 git 原语（自 git 2.5，2015 年起），在所有宿主上通用，且当新宿主出现时零维护。

### 声明意图，兜底为处方式

技能描述目标（"确保工作发生在隔离工作区"），并在原生工具可用时让位。它只对没有原生 worktree 支持的宿主以处方式方式给出具体 git 命令作为兜底。Step 1a 在前，并显式列出原生工具（`EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`）；Step 1b 在后，给出 git 兜底。原规格让 Step 1a 保持抽象（"you know your own toolkit"），但 TDD 证明当 Step 1a 太含糊时，代理会锚定到 Step 1b 的具体命令上。要使偏好可靠，必须显式命名工具，并加一个"同意即授权"的桥接。

### 基于来源的所有权

谁创建 worktree，谁就负责它的清理。如果宿主创建了它，superpowers 不动它。如果 superpowers 创建了它（通过 git 兜底），superpowers 负责清理。启发式：如果 worktree 位于 `.worktrees/` 或 `worktrees/` 之下，归 superpowers 所有。其他任何位置（`.claude/worktrees/`、`~/.codex/worktrees/`、`.gemini/worktrees/`，或旧的用户全局 Superpowers 路径）归宿主或你的搭档，保持原样。

## 设计

### 1. `using-git-worktrees` SKILL.md 重写

该技能在创建之前新增三步，并简化创建流程。

#### Step 0：检测既有隔离

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

三种结果：

| 条件 | 含义 | 动作 |
|-----------|---------|--------|
| `GIT_DIR == GIT_COMMON` | 普通仓库检出 | 进入 Step 0.5 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 已在链接型 worktree 中 | 跳到 Step 3（项目 setup）。报告："Already in isolated workspace at `<path>` on branch `<name>`." |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 外部管理的 worktree（例如 Codex App 沙箱） | 跳到 Step 3。报告："Already in isolated workspace at `<path>` (detached HEAD, externally managed)." |

Step 0 不关心谁创建了 worktree，也不关心是哪个宿主在运行。worktree 就是 worktree，不论来源。

**Submodule 守卫：** `GIT_DIR != GIT_COMMON` 在 git submodule 内部也为真。在下结论"已在一个 worktree 里"之前，先确认我们不是在一个 submodule 中：

```bash
# 如果这条返回一个路径，我们就在 submodule 中，而不是 worktree
git rev-parse --show-superproject-working-tree 2>/dev/null
```

如果是在 submodule 中，按 `GIT_DIR == GIT_COMMON` 处理（进入 Step 0.5）。

#### Step 0.5：同意

当 Step 0 未发现既有隔离（`GIT_DIR == GIT_COMMON`）时，先询问再创建：

> "Would you like me to set up an isolated worktree? This protects your current branch from changes. (y/n)"

如果同意，进入 Step 1。如果不同意，就地工作 —— 跳到 Step 3，不创建 worktree。

当 Step 0 检测到既有隔离时，完全跳过这一步（对已经存在的东西提问没有意义）。

#### Step 1a：原生工具（首选）

> 你的搭档已请求一个隔离工作区（Step 0 同意）。检查你的可用工具 —— 你是否有 `EnterWorktree`、`WorktreeCreate`、一条 `/worktree` 命令，或一个 `--worktree` flag？如果是：你的搭档同意创建 worktree 就是授权你去使用它。现在使用它，并跳到 Step 3。

使用原生工具后，跳到 Step 3（项目 setup）。

**设计注 —— TDD 修订：** 原规格使用一段刻意简短、抽象的 Step 1a（"You know your own toolkit — the skill does not need to name specific tools"）。TDD 验证推翻了这一点：代理锚定到 Step 1b 的具体 git 命令上，忽略抽象指导（2/6 通过率）。三项改动修复了它（GREEN 与 PRESSURE 测试合计 50/50 通过率）：

1. **显式工具命名** —— 按名字列出 `EnterWorktree`、`WorktreeCreate`、`/worktree`、`--worktree`，把决策从解释（"我是否有原生工具？"）转化为事实查找（"`EnterWorktree` 是否在我的工具列表里？"）。没有这些工具的平台上的代理只需检查、一无所获、落到 Step 1b。未观察到误报。
2. **同意桥接** —— "your搭档's consent to create a worktree is your authorization to use it" 直接回应 `EnterWorktree` 的工具级护栏（"ONLY when user explicitly asks"）。工具描述会覆盖技能指令（Claude Code #29950），所以技能必须把你的搭档的同意塑造成工具所要求的授权。
3. **Red Flag 条目** —— 在 Red Flags 节里点名这个具体的反模式（"Use `git worktree add` when you have a native worktree tool — this is the #1 mistake"）。

文件拆分（把 Step 1b 放进独立技能）已被测试证明不必要。锚定问题靠 Step 1a 文本的质量解决，而不是靠物理上把 git 命令分离出去。使用完整 240 行技能（所有 git 命令可见）的对照测试 20/20 通过。

#### Step 1b：Git Worktree 兜底

当没有原生工具可用时，手动创建 worktree。

**目录选择**（优先顺序）：
1. 检查项目的代理指令文件（CLAUDE.md、GEMINI.md、AGENTS.md、.cursorrules，或同等文件）中是否有 worktree 目录偏好。
2. 检查既有的 `.worktrees/` 或 `worktrees/` 目录 —— 若找到则使用它。若两者都存在，`.worktrees/` 胜出。
3. 默认为 `.worktrees/`。

不提供交互式目录选择提示。不检测或提供旧的用户全局 Superpowers worktree 路径；新的手动 worktree 默认为项目本地，除非你的搭档显式指定其他位置。

**安全校验**（仅限项目本地目录）：

```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未被忽略，先把它加入 `.gitignore` 并提交再继续。

**创建：**

```bash
git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**钩子感知：** Git worktree 不继承父仓库的 hooks 目录。通过 1b 创建 worktree 后，如果主仓库存在 hooks 目录，则把它符号链接过来：

```bash
if [ -d "$MAIN_ROOT/.git/hooks" ]; then
    ln -sf "$MAIN_ROOT/.git/hooks" "$path/.git/hooks"
fi
```

这能防止 pre-commit 检查、linter 与其他钩子在工作转移到 worktree 时静默失效。（想法来自 PR #965。）

**沙箱兜底：** 如果 `git worktree add` 因权限错误失败，视作受限环境。跳过创建、就地工作、进入 Step 3。

**步骤编号注：** 当前技能的 Steps 1-4 是一个扁平列表。本次重设计使用 0、0.5、1a、1b、3、4。没有 Step 2 —— 它曾是旧的、整块的 "Create Isolated Workspace"，现已拆成 1a/1b 结构。实现时应干净地重编号（例如 0 → "Step 0: Detect"、0.5 → 位于 Step 0 的流程内、1a/1b → "Step 1"、3 → "Step 2"、4 → "Step 3"），或保留当前编号并加一条说明。由实现者选择。

#### Steps 3-4：项目 Setup 与基线测试（不变）

无论哪条路径创建了工作区（Step 0 检测到既有、Step 1a 原生工具、Step 1b git 兜底，或根本没有 worktree），执行都汇聚到一起：

- **Step 3：** 自动检测并运行项目 setup（`npm install`、`cargo build`、`pip install`、`go mod download` 等）
- **Step 4：** 运行测试套件。如果测试失败，报告失败并询问是否继续。

### 2. `finishing-a-development-branch` SKILL.md 重写

finishing 技能新增环境检测，并修复三个 bug。

#### Step 1：验证测试（不变）

运行项目的测试套件。如果测试失败，停止。不要提供完成选项。

#### Step 1.5：检测环境（新增）

重新运行与创建时 Step 0 相同的检测：

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

三条路径：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 需要清理 |
| `GIT_DIR != GIT_COMMON`，命名分支 | 标准 4 选项 | 基于来源（见 Step 5） |
| `GIT_DIR != GIT_COMMON`，分离 HEAD | 缩减菜单：作为新分支 push + PR、保留原样、丢弃 | 无 merge 选项（无法从分离 HEAD merge） |

#### Step 2：确定 Base Branch（不变）

#### Step 3：呈现选项

**普通仓库与命名分支 worktree：**

1. 本地 merge 回 `<base-branch>`
2. push 并创建 Pull Request
3. 保留分支原样（稍后我自己处理）
4. 丢弃这些工作

**分离 HEAD：**

1. 作为新分支 push 并创建 Pull Request
2. 保留原样（稍后我自己处理）
3. 丢弃这些工作

#### Step 4：执行选择

**选项 1（本地 merge）：**

```bash
# 为 CWD 安全取主仓库根（Bug #238 修复）
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先 merge，成功后再移除任何东西
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>

# 仅在 merge 成功之后：移除 worktree，再删除分支（Bug #999 修复）
git worktree remove "$WORKTREE_PATH"  # 仅当 superpowers 拥有它时
git branch -d <feature-branch>
```

顺序至关重要：merge → 验证 → 移除 worktree → 删除分支。旧技能在移除 worktree 之前删除分支（这会失败，因为 worktree 仍然引用该分支）。先移除 worktree 的朴素修复也错 —— 如果 merge 随后失败，工作目录已经没了，改动也丢了。

**选项 2（创建 PR）：**

Push 分支，创建 PR。**不要**清理 worktree —— 你的搭档需要它做 PR 迭代。（Bug #940 修复：移除矛盾的 "Then: Cleanup worktree" 散文。）

**选项 3（保留原样）：** 无动作。

**选项 4（丢弃）：** 要求键入 "discard" 确认。然后移除 worktree（若 superpowers 拥有它），强制删除分支。

#### Step 5：清理（已更新）

```
if GIT_DIR == GIT_COMMON:
    # 普通仓库，无 worktree 要清理
    done

if worktree path is under .worktrees/ or worktrees/:
    # Superpowers 创建的 —— 清理归我们
    cd to main repo root       # Bug #238 修复
    git worktree remove <path>

else:
    # 宿主创建的 —— 不碰
    # 如果平台提供 workspace-exit 工具，使用它
    # 否则，把 worktree 留在原处
```

清理仅在选项 1 与选项 4 时运行。选项 2 与选项 3 总是保留 worktree。（Bug #940 修复。）

**陈旧 worktree 修剪：** 在任何 `git worktree remove` 之后，运行 `git worktree prune` 作为自愈步骤。worktree 目录可能被带外删除（例如被宿主清理、手动 `rm`、或 `.claude/` 清理），留下导致混乱错误的陈旧注册。一行，防止静默腐烂。（想法来自 PR #1072。）

### 3. Integration 更新

#### `subagent-driven-development` 与 `executing-plans`

两者当前都在其 integration 节把 `using-git-worktrees` 列为 REQUIRED。改为：

> `using-git-worktrees` — Ensures isolated workspace (creates one or verifies existing)

技能本身现在处理同意（Step 0.5）与检测（Step 0），所以调用方技能不需要再设门控或提示。

#### `writing-plans`

移除陈旧的说法 "should be run in a dedicated worktree (created by brainstorming skill)"。brainstorming 是一个设计技能，不创建 worktree。worktree 提示发生在执行阶段，通过 `using-git-worktrees`。

### 4. 平台中立的指令文件引用

worktree 相关技能中所有硬编码的 `CLAUDE.md` 实例都替换为：

> "your project's agent instruction file (CLAUDE.md, GEMINI.md, AGENTS.md, .cursorrules, or equivalent)"

这适用于 Step 1b 中的目录偏好检查。

## Bug 修复（打包）

| Bug | 问题 | 修复 | 位置 |
|-----|---------|-----|----------|
| #940 | 选项 2 的散文说 "Then: Cleanup worktree (Step 5)"，但快速参考说要保留。Step 5 说 "For Options 1, 2, 4"，但 Common Mistakes 说 "Options 1 and 4 only." | 从选项 2 移除清理。Step 5 仅适用于选项 1 与选项 4。 | finishing SKILL.md |
| #999 | 选项 1 在移除 worktree 之前删除分支。`git branch -d` 可能失败，因为 worktree 仍引用该分支。 | 重排为：merge → 验证测试 → 移除 worktree → 删除分支。merge 必须在移除任何东西之前成功。 | finishing SKILL.md |
| #238 | 如果 CWD 位于被移除的 worktree 内部，`git worktree remove` 静默失败。 | 增加 CWD 守卫：在 `git worktree remove` 之前 `cd` 到主仓库根。 | finishing SKILL.md |

## 已解决的 Issues

| Issue | 解决方式 |
|-------|-----------|
| #940 | 直接修复（Bug #940） |
| #991 | Step 0.5 的可选同意 |
| #918 | Step 0 检测 + Step 1.5 finishing 检测 |
| #1009 | 由 Step 1a 解决 —— 代理使用原生工具（例如 `EnterWorktree`），它创建在宿主原生路径。依赖 Step 1a 起作用；见风险。 |
| #999 | 直接修复（Bug #999） |
| #238 | 直接修复（Bug #238） |
| #1049 | 平台中立的指令文件引用 |
| #279 | 由 detect-and-defer 解决 —— 原生路径被尊重，因为我们不覆盖它们 |
| #574 | **推迟。** 本规格不碰 bug 所在的 brainstorming 技能。完整修复（在 brainstorming 清单中加入 worktree 步骤）属于 Phase 4。 |

## 风险

### Step 1a 是承重假设 —— 已解决

Step 1a —— 代理优先使用原生 worktree 工具而非 git 兜底 —— 是整个设计所依赖的基础。如果代理在有原生支持的宿主上忽略 Step 1a 并落到 Step 1b，detect-and-defer 就彻底失败。

**状态：** 此风险在实现期间显现。原始的抽象 Step 1a（"You know your own toolkit"）在 Claude Code 上 2/6 失败。TDD 门控按设计工作 —— 它在任何技能文件被修改之前就抓到了失败，避免了一次破损发布。三轮 REFACTOR 迭代识别了根因（代理锚定在具体命令上、工具描述的护栏覆盖了技能指令），并产出了在 GREEN 与 PRESSURE 测试上 50/50 验证通过的修复。详见上文 Step 1a 设计注。

**跨平台验证：**

截至 2026-04-06，Claude Code 是唯一拥有代理可在会话中途调用的 worktree 工具（`EnterWorktree`）的宿主。其他所有宿主要么在代理启动之前创建 worktree（Codex App、Gemini CLI、Cursor），要么没有原生 worktree 支持（Codex CLI、OpenCode）。Step 1a 是向前兼容的：当其他宿主添加可由代理调用的 worktree 工具时，代理会把它们与列出的示例匹配并使用它们，无需技能改动。

| 宿主 | 当前 worktree 模型 | 技能机制 | 已测 |
|---------|----------------------|-----------------|--------|
| Claude Code | 代理可调用 `EnterWorktree` | Step 1a | 50/50（GREEN + PRESSURE） |
| Codex CLI | 无原生工具（仅 shell） | Step 1b git 兜底 | 6/6（`codex exec`） |
| Gemini CLI | 启动时 `--worktree` flag，无代理工具 | 带 flag 时 Step 0，不带时 Step 1b | Step 0: 1/1, Step 1b: 1/1（`gemini -p`） |
| Cursor Agent | 面向你的搭档的 `/worktree`，无代理工具 | 你的搭档激活时 Step 0，否则 Step 1b | Step 0: 1/1, Step 1b: 1/1（`cursor-agent -p`） |
| Codex App | 平台管理，分离 HEAD，无代理工具 | Step 0 检测既有 | 1/1 模拟 |
| OpenCode | 仅检测（`ctx.worktree`），无代理工具 | Step 1b git 兜底 | 未测（无 CLI 访问） |

**残余风险：**
1. 如果 Anthropic 把 `EnterWorktree` 的工具描述改得更严（例如 "Do not use based on skill instructions"），同意桥接就会破裂。值得提一个 issue，请求工具描述容纳技能驱动的调用。
2. 当其他宿主添加可由代理调用的 worktree 工具时，它们可能使用不在 Step 1a 列表中的名字。列表应当随新工具出现而更新。通用的措辞（"a worktree or workspace-isolation tool"）提供了一些前向覆盖。

### 来源启发式

`.worktrees/` 或 `worktrees/` = 我们的，其他一切 = 不碰的启发式对每个当前宿主都有效。如果某个未来宿主采用这些项目本地目录之一作为其约定，就会有误报（superpowers 试图清理一个宿主拥有的 worktree）。同样，如果某个你的搭档在没有 superpowers 的情况下手动运行 `git worktree add .worktrees/experiment`，我们会错误地声称所有权。两者都是低风险 —— 每个宿主都使用带品牌色的路径，手动创建 `.worktrees/` 也不太可能 —— 但值得记录。

### 分离 HEAD 的 finishing

对分离 HEAD worktree 的缩减菜单（无 merge 选项）对 Codex App 的沙箱模型是正确的。如果你的搭档因其他原因处于分离 HEAD，缩减菜单仍然合理 —— 你确实无法在不先创建分支的情况下从分离 HEAD merge。

## 实现注

两个技能文件都含核心步骤之外的章节，需要在实现期间更新：

- **Frontmatter**（`name`、`description`）：更新以反映 detect-and-defer 行为
- **Quick Reference 表**：重写以匹配新步骤结构与 bug 修复
- **Common Mistakes 节**：更新或删除引用旧行为的条目（例如 "Skip CLAUDE.md check" 现在是错的）
- **Red Flags 节**：更新以反映新的优先级（例如 "Never create a worktree when Step 0 detects existing isolation"）
- **Integration 节**：更新技能之间的交叉引用

本规格描述*改了什么*；实现计划会指定对这些次要章节的精确编辑。

## 未来工作（不在本规格内）

- **Phase 3 剩余项：** `$TMPDIR` 目录选项（#666）、缓存与环境继承的 setup 文档（#299）
- **Phase 4：** 用于路径强制的 PreToolUse 钩子（#1040）、每个 worktree 的 env 约定（#597）、brainstorming 清单的 worktree 步骤（#574）、多仓库文档（#710）
