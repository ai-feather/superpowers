---
name: finishing-a-development-branch
description: 当实现完成、所有测试通过、且你需要决定如何集成工作时使用——通过呈现合并、PR 或清理的结构化选项来指导开发工作的完成
---

# 完成开发分支

## 概览

通过呈现清晰选项并处理所选工作流，指导开发工作的完成。

**核心原则：** 验证测试 → 检测环境 → 呈现选项 → 执行选择 → 清理。

**开始时宣告：** "我在用 finishing-a-development-branch 技能来完成这项工作。"

## 流程

### 步骤 1：验证测试

**在呈现选项之前，验证测试通过：**

```bash
# 运行项目的测试套件
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

停下。不要进入步骤 2。

**如果测试通过：** 继续步骤 2。

### 步骤 2：检测环境

**在呈现选项之前确定工作区状态：**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

这决定显示哪个菜单以及清理如何工作：

| 状态 | 菜单 | 清理 |
|-------|------|---------|
| `GIT_DIR == GIT_COMMON`（普通仓库） | 标准 4 选项 | 无 worktree 要清理 |
| `GIT_DIR != GIT_COMMON`，有名分支 | 标准 4 选项 | 基于来源（见步骤 6） |
| `GIT_DIR != GIT_COMMON`，detached HEAD | 缩减 3 选项（无合并） | 无清理（外部管理） |

### 步骤 3：确定基分支

```bash
# 尝试常见的基分支
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

或问："这个分支从 main 分出——对吗？"

### 步骤 4：呈现选项

**普通仓库和有名分支 worktree——精确呈现这 4 个选项：**

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

**Detached HEAD——精确呈现这 3 个选项：**

```
Implementation complete. You're on a detached HEAD (externally managed workspace).

1. Push as new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

**不要加解释**——保持选项简洁。

### 步骤 5：执行选择

#### 选项 1：本地合并

```bash
# 取主仓库根以保证 CWD 安全
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"

# 先合并——在移除任何东西之前验证成功
git checkout <base-branch>
git pull
git merge <feature-branch>

# 在合并结果上验证测试
<test command>

# 只有合并成功后：清理 worktree（步骤 6），然后删除分支
```

然后：清理 worktree（步骤 6），然后删除分支：

```bash
git branch -d <feature-branch>
```

#### 选项 2：推送并创建 PR

```bash
# 推送分支
git push -u origin <feature-branch>
```

**不要清理 worktree**——用户需要它活着以迭代 PR 反馈。

#### 选项 3：保持原样

报告："保留分支 <name>。Worktree 保留在 <path>。"

**不要清理 worktree。**

#### 选项 4：丢弃

**先确认：**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

等待确切确认。

如果确认：
```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
```

然后：清理 worktree（步骤 6），然后强制删除分支：
```bash
git branch -D <feature-branch>
```

### 步骤 6：清理工作区

**仅对选项 1 和 4 运行。** 选项 2 和 3 总是保留 worktree。

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

**如果 `GIT_DIR == GIT_COMMON`：** 普通仓库，无 worktree 要清理。完成。

**如果 worktree 路径在 `.worktrees/` 或 `worktrees/` 下：** Superpowers 创建了这个 worktree——清理归我们管。

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune  # 自愈：清理任何陈旧的注册
```

**否则：** 宿主环境（harness）拥有这个工作区。不要移除它。如果你的平台提供工作区退出工具，用它。否则，让工作区留在原处。

## 快速参考

| 选项 | 合并 | 推送 | 保留 Worktree | 清理分支 |
|--------|-------|------|---------------|----------------|
| 1. 本地合并 | 是 | - | - | 是 |
| 2. 创建 PR | - | 是 | 是 | - |
| 3. 保持原样 | - | - | 是 | - |
| 4. 丢弃 | - | - | - | 是（强制） |

## 常见错误

**跳过测试验证**
- **问题：** 合并坏代码，创建失败 PR
- **修正：** 提供选项前总是验证测试

**开放式问题**
- **问题：** "我下一步该做什么？"含糊
- **修正：** 精确呈现 4 个结构化选项（detached HEAD 为 3 个）

**为选项 2 清理 worktree**
- **问题：** 移除用户 PR 迭代需要的 worktree
- **修正：** 仅对选项 1 和 4 清理

**移除 worktree 前删除分支**
- **问题：** `git branch -d` 失败，因为 worktree 仍引用该分支
- **修正：** 先合并，移除 worktree，然后删除分支

**从 worktree 内部运行 git worktree remove**
- **问题：** 当 CWD 在被移除的 worktree 内时命令静默失败
- **修正：** 在 `git worktree remove` 前总是 `cd` 到主仓库根

**清理 harness 拥有的 worktree**
- **问题：** 移除 harness 创建的 worktree 导致幽灵状态
- **修正：** 只清理 `.worktrees/` 或 `worktrees/` 下的 worktree

**丢弃无确认**
- **问题：** 意外删除工作
- **修正：** 要求键入"discard"确认

## 红旗

**绝不：**
- 带着失败的测试继续
- 不验证结果上的测试就合并
- 无确认就删除工作
- 无明确请求就强制推送
- 确认合并成功前移除 worktree
- 清理你没创建的 worktree（来源检查）
- 从 worktree 内部运行 `git worktree remove`

**总是：**
- 提供选项前验证测试
- 呈现菜单前检测环境
- 精确呈现 4 个选项（detached HEAD 为 3 个）
- 选项 4 要键入确认
- 仅对选项 1 和 4 清理 worktree
- worktree 移除前 `cd` 到主仓库根
- 移除后运行 `git worktree prune`
