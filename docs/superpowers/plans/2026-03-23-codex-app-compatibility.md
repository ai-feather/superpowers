# Codex App 兼容性实现计划

> **致代理工作者：** 必须使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来按任务实现本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 让 `using-git-worktrees`、`finishing-a-development-branch` 及相关技能在 Codex App 的沙箱化 worktree 环境中工作，同时不破坏既有行为。

**架构：** 在两个技能开头进行只读环境检测（`git-dir` vs `git-common-dir`）。如果已在 linked worktree 中，则跳过创建。如果在 detached HEAD 上，则输出交接载荷（handoff payload）而非 4 选项菜单。沙箱兜底捕获 worktree 创建期间的权限错误。

**技术栈：** Git、Markdown（技能文件是指令文档，不是可执行代码）

**规格：** `docs/superpowers/specs/2026-03-23-codex-app-compatibility-design.md`

---

## 文件结构

| 文件 | 职责 | 操作 |
|---|---|---|
| `skills/using-git-worktrees/SKILL.md` | worktree 创建 + 隔离 | 新增 Step 0 检测 + 沙箱兜底 |
| `skills/finishing-a-development-branch/SKILL.md` | 分支收尾工作流 | 新增 Step 1.5 检测 + 清理守卫 |
| `skills/subagent-driven-development/SKILL.md` | 用子代理执行计划 | 更新 Integration 说明 |
| `skills/executing-plans/SKILL.md` | 内联执行计划 | 更新 Integration 说明 |
| `skills/using-superpowers/references/codex-tools.md` | Codex 平台参考 | 新增检测 + 收尾文档 |

---

### Task 1: 为 `using-git-worktrees` 新增 Step 0

**文件：**
- Modify: `skills/using-git-worktrees/SKILL.md:14-15`（在 Overview 之后、Directory Selection Process 之前插入）

- [ ] **Step 1: 阅读当前技能文件**

完整阅读 `skills/using-git-worktrees/SKILL.md`。确定精确插入点：在 "Announce at start" 行（第 14 行）之后、`## Directory Selection Process`（第 16 行）之前。

- [ ] **Step 2: 插入 Step 0 小节**

在 Overview 小节与 `## Directory Selection Process` 之间插入以下内容：

```markdown
## Step 0: Check if Already in an Isolated Workspace

Before creating a worktree, check if one already exists:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**If `GIT_DIR` differs from `GIT_COMMON`:** You are already inside a linked worktree (created by the Codex App, Claude Code's Agent tool, a previous skill run, or the user). Do NOT create another worktree. Instead:

1. Run project setup (auto-detect package manager as in "Run Project Setup" below)
2. Verify clean baseline (run tests as in "Verify Clean Baseline" below)
3. Report with branch state:
   - On a branch: "Already in an isolated workspace at `<path>` on branch `<name>`. Tests passing. Ready to implement."
   - Detached HEAD: "Already in an isolated workspace at `<path>` (detached HEAD, externally managed). Tests passing. Note: branch creation needed at finish time. Ready to implement."

After reporting, STOP. Do not continue to Directory Selection or Creation Steps.

**If `GIT_DIR` equals `GIT_COMMON`:** Proceed with the full worktree creation flow below.

**Sandbox fallback:** If you proceed to Creation Steps but `git worktree add -b` fails with a permission error (e.g., "Operation not permitted"), treat this as a late-detected restricted environment. Fall back to the behavior above — run setup and baseline tests in the current directory, report accordingly, and STOP.
```

- [ ] **Step 3: 验证插入**

再次阅读文件。确认：
- Step 0 出现在 Overview 与 Directory Selection Process 之间
- 文件其余部分（Directory Selection、Safety Verification、Creation Steps 等）未变
- 无重复小节或破损的 markdown

- [ ] **Step 4: 提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "feat(using-git-worktrees): add Step 0 environment detection (PRI-823)

Skip worktree creation when already in a linked worktree. Includes
sandbox fallback for permission errors on git worktree add."
```

---

### Task 2: 更新 `using-git-worktrees` 的 Integration 小节

**文件：**
- Modify: `skills/using-git-worktrees/SKILL.md:211-215`（Integration > Called by）

- [ ] **Step 1: 更新三条 "Called by" 条目**

将第 212-214 行从：

```markdown
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
```

改为：

```markdown
- **brainstorming** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **subagent-driven-development** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
- **executing-plans** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2: 验证 Integration 小节**

阅读 Integration 小节。确认三条条目均已更新，"Pairs with" 未变。

- [ ] **Step 3: 提交**

```bash
git add skills/using-git-worktrees/SKILL.md
git commit -m "docs(using-git-worktrees): update Integration descriptions (PRI-823)

Clarify that skill ensures a workspace exists, not that it always creates one."
```

---

### Task 3: 为 `finishing-a-development-branch` 新增 Step 1.5

**文件：**
- Modify: `skills/finishing-a-development-branch/SKILL.md:38`（在 Step 1 之后、Step 2 之前插入）

- [ ] **Step 1: 阅读当前技能文件**

完整阅读 `skills/finishing-a-development-branch/SKILL.md`。确定精确插入点：在 "**If tests pass:** Continue to Step 2."（第 38 行）之后、`### Step 2: Determine Base Branch`（第 40 行）之前。

- [ ] **Step 2: 插入 Step 1.5 小节**

在 Step 1 与 Step 2 之间插入以下内容：

```markdown
### Step 1.5: Detect Environment

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Path A — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` is empty (externally managed worktree, detached HEAD):**

First, ensure all work is staged and committed (`git add` + `git commit`).

Then present this to the user (do NOT present the 4-option menu):

```
Implementation complete. All tests passing.
Current HEAD: <full-commit-sha>

This workspace is externally managed (detached HEAD).
I cannot create branches, push, or open PRs from here.

⚠ These commits are on a detached HEAD. If you do not create a branch,
they may be lost when this workspace is cleaned up.

If your host application provides these controls:
- "Create branch" — to name a branch, then commit/push/PR
- "Hand off to local" — to move changes to your local checkout

Suggested branch name: <ticket-id/short-description>
Suggested commit message: <summary-of-work>
```

Branch name: use ticket ID if available (e.g., `pri-823/codex-compat`), otherwise slugify the first 5 words of the plan title, otherwise omit. Avoid sensitive content in branch names.

Skip to Step 5 (cleanup is a no-op — see guard below).

**Path B — `GIT_DIR` differs from `GIT_COMMON` AND `BRANCH` exists (externally managed worktree, named branch):**

Proceed to Step 2 and present the 4-option menu as normal.

**Path C — `GIT_DIR` equals `GIT_COMMON` (normal environment):**

Proceed to Step 2 and present the 4-option menu as normal.
```

- [ ] **Step 3: 验证插入**

再次阅读文件。确认：
- Step 1.5 出现在 Step 1 与 Step 2 之间
- Step 2-5 未变
- Path A 的交接载荷包含 commit SHA 和数据丢失警告
- Path B 和 Path C 正常进入 Step 2

- [ ] **Step 4: 提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 1.5 environment detection (PRI-823)

Detect externally managed worktrees with detached HEAD and emit handoff
payload instead of 4-option menu. Includes commit SHA and data loss warning."
```

---

### Task 4: 为 `finishing-a-development-branch` 新增 Step 5 清理守卫

**文件：**
- Modify: `skills/finishing-a-development-branch/SKILL.md`（Step 5: Cleanup Worktree——按小节标题查找，Task 3 之后行号已偏移）

- [ ] **Step 1: 阅读当前 Step 5 小节**

在 `skills/finishing-a-development-branch/SKILL.md` 中找到 `### Step 5: Cleanup Worktree` 小节（Task 3 插入后行号已偏移）。当前 Step 5 为：

```markdown
### Step 5: Cleanup Worktree

**For Options 1, 2, 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

- [ ] **Step 2: 在既有逻辑之前新增清理守卫**

将 Step 5 小节替换为：

```markdown
### Step 5: Cleanup Worktree

**First, check if worktree is externally managed:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

If `GIT_DIR` differs from `GIT_COMMON`: skip worktree removal — the host environment owns this workspace.

**Otherwise, for Options 1 and 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.
```

注：原文写的是 "For Options 1, 2, 4"，但 Quick Reference 表和 Common Mistakes 小节写的是 "Options 1 & 4 only"。本次编辑让 Step 5 与那些小节对齐。

- [ ] **Step 3: 验证替换**

阅读 Step 5。确认：
- 清理守卫（重新检测）出现在最前
- 为非外部托管的 worktree 保留既有移除逻辑
- "Options 1 and 4"（不是 "1, 2, 4"）与 Quick Reference 和 Common Mistakes 一致

- [ ] **Step 4: 提交**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat(finishing-a-development-branch): add Step 5 cleanup guard (PRI-823)

Re-detect externally managed worktree at cleanup time and skip removal.
Also fixes pre-existing inconsistency: cleanup now correctly says
Options 1 and 4 only, matching Quick Reference and Common Mistakes."
```

---

### Task 5: 更新 `subagent-driven-development` 和 `executing-plans` 中的 Integration 行

**文件：**
- Modify: `skills/subagent-driven-development/SKILL.md:268`
- Modify: `skills/executing-plans/SKILL.md:68`

- [ ] **Step 1: 更新 `subagent-driven-development`**

将第 268 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 2: 更新 `executing-plans`**

将第 68 行从：
```
- **superpowers:using-git-worktrees** - REQUIRED: Set up isolated workspace before starting
```
改为：
```
- **superpowers:using-git-worktrees** - REQUIRED: Ensures isolated workspace (creates one or verifies existing)
```

- [ ] **Step 3: 验证两个文件**

阅读 `skills/subagent-driven-development/SKILL.md` 第 268 行和 `skills/executing-plans/SKILL.md` 第 68 行。确认两者都写的是 "Ensures isolated workspace (creates one or verifies existing)"。

- [ ] **Step 4: 提交**

```bash
git add skills/subagent-driven-development/SKILL.md skills/executing-plans/SKILL.md
git commit -m "docs(sdd, executing-plans): update worktree Integration descriptions (PRI-823)

Clarify that using-git-worktrees ensures a workspace exists rather than
always creating one."
```

---

### Task 6: 为 `codex-tools.md` 新增环境检测文档

**文件：**
- Modify: `skills/using-superpowers/references/codex-tools.md:25`（在末尾追加）

- [ ] **Step 1: 阅读当前文件**

完整阅读 `skills/using-superpowers/references/codex-tools.md`。确认它在 multi_agent 小节之后于第 25-26 行结束。

- [ ] **Step 2: 追加两个新小节**

在文件末尾新增：

```markdown

## Environment Detection

Skills that create worktrees or finish branches should detect their
environment with read-only git commands before proceeding:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

- `GIT_DIR != GIT_COMMON` → already in a linked worktree (skip creation)
- `BRANCH` empty → detached HEAD (cannot branch/push/PR from sandbox)

See `using-git-worktrees` Step 0 and `finishing-a-development-branch`
Step 1.5 for how each skill uses these signals.

## Codex App Finishing

When the sandbox blocks branch/push operations (detached HEAD in an
externally managed worktree), the agent commits all work and informs
the user to use the App's native controls:

- **"Create branch"** — names the branch, then commit/push/PR via App UI
- **"Hand off to local"** — transfers work to the user's local checkout

The agent can still run tests, stage files, and output suggested branch
names, commit messages, and PR descriptions for the user to copy.
```

- [ ] **Step 3: 验证新增内容**

完整阅读文件。确认：
- 两个新小节出现在既有内容之后
- Bash 代码块正确渲染（未被转义）
- 对 Step 0 和 Step 1.5 的交叉引用存在

- [ ] **Step 4: 提交**

```bash
git add skills/using-superpowers/references/codex-tools.md
git commit -m "docs(codex-tools): add environment detection and App finishing docs (PRI-823)

Document the git-dir vs git-common-dir detection pattern and the Codex
App's native finishing flow for skills that need to adapt."
```

---

### Task 7: 自动化测试 — 环境检测

**文件：**
- Create: `tests/codex-app-compat/test-environment-detection.sh`

- [ ] **Step 1: 创建测试目录**

```bash
mkdir -p tests/codex-app-compat
```

- [ ] **Step 2: 编写检测测试脚本**

创建 `tests/codex-app-compat/test-environment-detection.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# Test environment detection logic from PRI-823
# Tests the git-dir vs git-common-dir comparison used by
# using-git-worktrees Step 0 and finishing-a-development-branch Step 1.5

PASS=0
FAIL=0
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

log_pass() { echo "  PASS: $1"; PASS=$((PASS + 1)); }
log_fail() { echo "  FAIL: $1"; FAIL=$((FAIL + 1)); }

# Helper: run detection and return "linked" or "normal"
detect_worktree() {
  local git_dir git_common
  git_dir=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
  git_common=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
  if [ "$git_dir" != "$git_common" ]; then
    echo "linked"
  else
    echo "normal"
  fi
}

echo "=== Test 1: Normal repo detection ==="
cd "$TEMP_DIR"
git init test-repo > /dev/null 2>&1
cd test-repo
git commit --allow-empty -m "init" > /dev/null 2>&1
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Normal repo detected as normal"
else
  log_fail "Normal repo detected as '$result' (expected 'normal')"
fi

echo "=== Test 2: Linked worktree detection ==="
git worktree add "$TEMP_DIR/test-wt" -b test-branch > /dev/null 2>&1
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Linked worktree detected as linked"
else
  log_fail "Linked worktree detected as '$result' (expected 'linked')"
fi

echo "=== Test 3: Detached HEAD detection ==="
git checkout --detach HEAD > /dev/null 2>&1
branch=$(git branch --show-current)
if [ -z "$branch" ]; then
  log_pass "Detached HEAD: branch is empty"
else
  log_fail "Detached HEAD: branch is '$branch' (expected empty)"
fi

echo "=== Test 4: Linked worktree + detached HEAD (Codex App simulation) ==="
result=$(detect_worktree)
branch=$(git branch --show-current)
if [ "$result" = "linked" ] && [ -z "$branch" ]; then
  log_pass "Codex App simulation: linked + detached HEAD"
else
  log_fail "Codex App simulation: result='$result', branch='$branch'"
fi

echo "=== Test 5: Cleanup guard — linked worktree should NOT remove ==="
cd "$TEMP_DIR/test-wt"
result=$(detect_worktree)
if [ "$result" = "linked" ]; then
  log_pass "Cleanup guard: linked worktree correctly detected (would skip removal)"
else
  log_fail "Cleanup guard: expected 'linked', got '$result'"
fi

echo "=== Test 6: Cleanup guard — main repo SHOULD remove ==="
cd "$TEMP_DIR/test-repo"
result=$(detect_worktree)
if [ "$result" = "normal" ]; then
  log_pass "Cleanup guard: main repo correctly detected (would proceed with removal)"
else
  log_fail "Cleanup guard: expected 'normal', got '$result'"
fi

# Cleanup worktree before temp dir removal
cd "$TEMP_DIR/test-repo"
git worktree remove "$TEMP_DIR/test-wt" > /dev/null 2>&1 || true

echo ""
echo "=== Results: $PASS passed, $FAIL failed ==="
if [ "$FAIL" -gt 0 ]; then
  exit 1
fi
```

- [ ] **Step 3: 赋予可执行权限并运行**

```bash
chmod +x tests/codex-app-compat/test-environment-detection.sh
./tests/codex-app-compat/test-environment-detection.sh
```

预期输出：6 passed, 0 failed.

- [ ] **Step 4: 提交**

```bash
git add tests/codex-app-compat/test-environment-detection.sh
git commit -m "test: add environment detection tests for Codex App compat (PRI-823)

Tests git-dir vs git-common-dir comparison in normal repo, linked
worktree, detached HEAD, and cleanup guard scenarios."
```

---

### Task 8: 最终验证

**文件：**
- Read: 所有 5 个被修改的技能文件

- [ ] **Step 1: 运行自动化检测测试**

```bash
./tests/codex-app-compat/test-environment-detection.sh
```

预期：6 passed, 0 failed.

- [ ] **Step 2: 阅读每个修改文件并验证改动**

端到端阅读每个文件：
- `skills/using-git-worktrees/SKILL.md` — Step 0 存在，其余未变
- `skills/finishing-a-development-branch/SKILL.md` — Step 1.5 存在，清理守卫存在，其余未变
- `skills/subagent-driven-development/SKILL.md` — 第 268 行已更新
- `skills/executing-plans/SKILL.md` — 第 68 行已更新
- `skills/using-superpowers/references/codex-tools.md` — 末尾有两个新小节

- [ ] **Step 3: 验证无意外改动**

```bash
git diff --stat HEAD~7
```

应显示恰好 6 个文件被改动（5 个技能文件 + 1 个测试文件）。无其他文件被修改。

- [ ] **Step 4: 运行既有测试套件**

如果存在测试运行器：
```bash
# Run skill-triggering tests
# Note: tests/skill-triggering/ was lifted into drill scenarios on 2026-05-06.
# See evals/scenarios/triggering-*.yaml. The reference below is a dated artifact.
./tests/skill-triggering/run-all.sh 2>/dev/null || echo "Skill triggering tests not available in this environment"

# Run SDD integration test
./tests/claude-code/test-subagent-driven-development-integration.sh 2>/dev/null || echo "SDD integration test not available in this environment"
```

注：这些测试需要带 `--dangerously-skip-permissions` 的 Claude Code。如果不可用，记录回归测试应手动运行。
