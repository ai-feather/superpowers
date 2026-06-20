---
name: using-git-worktrees
description: 当开始需要与当前工作区隔离的功能工作，或执行实现计划之前使用——通过原生工具或 git worktree 回退确保存在一个隔离工作区
---

# 使用 Git Worktree

## 概览

确保工作发生在隔离工作区。优先用你平台的原生 worktree 工具。仅在没有原生工具可用时回退到手动 git worktree。

**核心原则：** 先检测既有隔离。然后用原生工具。然后回退到 git。永远不要与 harness 对抗。

**开始时宣告：** "我在用 using-git-worktrees 技能来设置隔离工作区。"

## 步骤 0：检测既有隔离

**在创建任何东西之前，检查你是否已经在一个隔离工作区里。**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**子模块守卫：** `GIT_DIR != GIT_COMMON` 在 git 子模块内也为真。在得出"已经在 worktree 里"的结论之前，验证你不在子模块里：

```bash
# 如果它返回一个路径，你在子模块里，不是 worktree——当作普通仓库处理
git rev-parse --show-superproject-working-tree 2>/dev/null
```

**如果 `GIT_DIR != GIT_COMMON`（且不是子模块）：** 你已经在一个链接的 worktree 里。跳到步骤 2（项目设置）。不要创建另一个 worktree。

带分支状态报告：
- 在分支上："已经在隔离工作区 `<path>`，分支 `<name>`。"
- Detached HEAD："已经在隔离工作区 `<path>`（detached HEAD，外部管理）。分支创建需要在完成时进行。"

**如果 `GIT_DIR == GIT_COMMON`（或在子模块里）：** 你在普通仓库检出里。

用户是否已在你的指令中表明其 worktree 偏好？如果没有，在创建 worktree 前请求同意：

> "你想让我设置一个隔离的 worktree 吗？它会保护你当前分支免受改动。"

遵从任何既有的声明偏好，无需询问。如果用户拒绝同意，原地工作并跳到步骤 2。

## 步骤 1：创建隔离工作区

**你有两种机制。按此顺序尝试。**

### 1a. 原生 Worktree 工具（首选）

用户已请求隔离工作区（步骤 0 同意）。你是否已有创建 worktree 的方式？它可能是一个叫 `EnterWorktree`、`WorktreeCreate`、`/worktree` 命令，或 `--worktree` flag 的工具。如果有，用它并跳到步骤 2。

原生工具自动处理目录放置、分支创建和清理。当你有原生工具时用 `git worktree add` 会创建你的 harness 看不到也管不了的幽灵状态。

仅在你没有可用的原生 worktree 工具时才进入步骤 1b。

### 1b. Git Worktree 回退

**仅当步骤 1a 不适用时使用** —— 你没有可用的原生 worktree 工具。用 git 手动创建 worktree。

#### 目录选择

遵循此优先顺序。显式的用户偏好总是胜过观察到的文件系统状态。

1. **检查你的指令里是否有声明的 worktree 目录偏好。** 如果用户已指定一个，无需询问就用它。

2. **检查既有的项目本地 worktree 目录：**
   ```bash
   ls -d .worktrees 2>/dev/null     # 首选（隐藏）
   ls -d worktrees 2>/dev/null      # 备选
   ```
   如果找到，用它。如果两者都在，`.worktrees` 胜出。

3. **如果没有其他指引可用**，默认用项目根的 `.worktrees/`。

#### 安全验证（仅项目本地目录）

**必须在创建 worktree 前验证目录被忽略：**

```bash
git check-ignore -q .worktrees 2>/dev/null || git check-ignore -q worktrees 2>/dev/null
```

**如果未被忽略：** 加到 .gitignore，提交改动，然后继续。

**为什么关键：** 防止意外把 worktree 内容提交到仓库。

#### 创建 Worktree

```bash
# 根据所选位置确定路径
path="$LOCATION/$BRANCH_NAME"

git worktree add "$path" -b "$BRANCH_NAME"
cd "$path"
```

**沙箱回退：** 如果 `git worktree add` 因权限错误失败（沙箱拒绝），告诉用户沙箱阻止了 worktree 创建，你改为在当前目录工作。然后原地运行 setup 和基线测试。

## 步骤 2：项目设置

自动检测并运行合适的 setup：

```bash
# Node.js
if [ -f package.json ]; then npm install; fi

# Rust
if [ -f Cargo.toml ]; then cargo build; fi

# Python
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi

# Go
if [ -f go.mod ]; then go mod download; fi
```

## 步骤 3：验证干净基线

运行测试以确保工作区干净起步：

```bash
# 用项目合适的命令
npm test / cargo test / pytest / go test ./...
```

**如果测试失败：** 报告失败，问是继续还是调查。

**如果测试通过：** 报告就绪。

### 报告

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## 快速参考

| 情况 | 动作 |
|-----------|--------|
| 已在链接的 worktree 里 | 跳过创建（步骤 0） |
| 在子模块里 | 当作普通仓库（步骤 0 守卫） |
| 原生 worktree 工具可用 | 用它（步骤 1a） |
| 无原生工具 | Git worktree 回退（步骤 1b） |
| `.worktrees/` 存在 | 用它（验证被忽略） |
| `worktrees/` 存在 | 用它（验证被忽略） |
| 两者都在 | 用 `.worktrees/` |
| 都不在 | 检查指令文件，然后默认 `.worktrees/` |
| 目录未被忽略 | 加到 .gitignore + 提交 |
| 创建时权限错误 | 沙箱回退，原地工作 |
| 基线时测试失败 | 报告失败 + 询问 |
| 无 package.json/Cargo.toml | 跳过依赖安装 |

## 常见错误

### 与 harness 对抗

- **问题：** 当平台已提供隔离时用 `git worktree add`
- **修正：** 步骤 0 检测既有隔离。步骤 1a 遵从原生工具。

### 跳过检测

- **问题：** 在既有 worktree 内创建嵌套 worktree
- **修正：** 创建任何东西前总是运行步骤 0

### 跳过忽略验证

- **问题：** Worktree 内容被跟踪，污染 git status
- **修正：** 创建项目本地 worktree 前总是用 `git check-ignore`

### 假设目录位置

- **问题：** 造成不一致，违反项目约定
- **修正：** 遵循优先级：显式指令 > 既有项目本地目录 > 默认

### 带着失败测试继续

- **问题：** 无法区分新 bug 与既有问题
- **修正：** 报告失败，获得明确许可才继续

## 红旗

**绝不：**
- 当步骤 0 检测到既有隔离时创建 worktree
- 当你有原生 worktree 工具（例如 `EnterWorktree`）时用 `git worktree add`。这是头号错误——如果你有它，就用它。
- 通过直接跳到步骤 1b 的 git 命令跳过步骤 1a
- 未验证被忽略就创建 worktree（项目本地）
- 跳过基线测试验证
- 不询问就带着失败测试继续

**总是：**
- 先运行步骤 0 检测
- 优先原生工具而非 git 回退
- 遵循目录优先级：显式指令 > 既有项目本地目录 > 默认
- 项目本地验证目录被忽略
- 自动检测并运行项目 setup
- 验证干净的测试基线
