# 将 drill 提升为 superpowers 的 `evals/` —— 实现计划

> **给代理工作者：** 必用子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐项实现本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将独立的 `obra/drill` 技能合规性基准测试移入 superpowers 的顶层 `evals/` 目录；在子代理逐文件验证 drill 场景覆盖情况后，删除 `superpowers/tests/` 下冗余的 bash 测试；更新顶层文档，确保贡献者能直接对接新结构。

**架构：** 在新分支 `f/evals-lift` 上对 `dev` 发起单个 PR。drill 源码按原样复制，使用显式的 rsync excludes 把 `.git/`、`.venv/` 等排除在新目录之外。`drill/cli.py` 中加一个小助手，把 `SUPERPOWERS_ROOT` 默认指向 `evals/` 的父目录，贡献者无需再手动设置环境变量。每条 bash 测试的删除都由一个子代理把关——它会比较该 bash 测试的断言与它所声明对应的 drill 场景的 verify 块。计划文档和发布说明中的历史引用只做注释，不重写。

**技术栈：** Python 3.11 + uv（drill 现有工具链，保持不变）；rsync；bash；git。

**规格：** `docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md` —— 请先读它。

**drill 源码位置：** `/Users/jesse/Documents/GitHub/superpowers/drill/`（与 `superpowers/` 同级）。

---

## Task 1: 基于 dev 创建分支

**文件：** 无（仅 git 操作）

- [ ] **Step 1: 确认工作树干净**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git status --short
```

预期：输出为空（或只有未跟踪的 `.opencode/package-lock.json`，这没问题）。

- [ ] **Step 2: 拉取最新 dev**

```bash
git fetch origin dev:dev
```

- [ ] **Step 3: 创建分支**

```bash
git checkout -b f/evals-lift dev
```

预期：`Switched to a new branch 'f/evals-lift'`。

- [ ] **Step 4: 健全性检查**

```bash
git log --oneline -1
```

预期输出以 `origin/dev` 当前指向的 commit 开头（当前为 `b4363df docs: turned the dash in "- Jesse" into an escape sequence (#1474)`）。

---

## Task 2: 复制时记录 drill 的 SHA

**文件：** 无（记录该值，供 lift commit 消息使用）

- [ ] **Step 1: 获取 drill 当前的 HEAD sha**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
DRILL_SHA=$(git rev-parse HEAD)
echo "$DRILL_SHA"
```

- [ ] **Step 2: 确认 drill 没有未提交的改动**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
git status --short
```

预期：为空（无未跟踪或已修改的文件）。如果输出非空，停止并报告——lift 前 drill 工作树必须干净，否则 SHA 固定毫无意义。

- [ ] **Step 3: 把 SHA 存进 shell 环境变量，供下一个任务使用**

```bash
echo "DRILL_SHA=$DRILL_SHA"  # 记下来供 Task 3 使用
```

---

## Task 3: 用 rsync 把 drill 同步进 evals/

**文件：**
- 创建：`evals/`（drill 的整棵目录树，减去 excludes）

- [ ] **Step 1: 确认源路径和目标路径**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
test -d /Users/jesse/Documents/GitHub/superpowers/drill && echo "drill source: OK"
test ! -d evals && echo "evals/ does not yet exist: OK"
```

预期：两条 echo 都打印。

- [ ] **Step 2: 用显式 excludes 把 drill rsync 到 evals/**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
rsync -a \
  --exclude=.git \
  --exclude=.venv \
  --exclude=results \
  --exclude=.env \
  --exclude=__pycache__ \
  --exclude='*.egg-info' \
  --exclude=.private-journal \
  --exclude='*.pyc' \
  /Users/jesse/Documents/GitHub/superpowers/drill/ \
  evals/
```

- [ ] **Step 3: 确认 excludes 生效**

```bash
find evals -name '.git' -type d
find evals -name '.venv' -type d
find evals -name 'results' -type d
find evals -name '.env'
find evals -name '__pycache__' -type d
find evals -name '*.egg-info' -type d
```

预期：每条命令都没有输出。如果任何一条返回了路径，继续之前手动 `rm -rf` 掉它。

- [ ] **Step 4: 为 commit 消息再次确认源 SHA**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
DRILL_SHA=$(git rev-parse HEAD)
echo "$DRILL_SHA"
```

预期：与 Task 2 step 1 中的 SHA 一致。

- [ ] **Step 5: 全部 stage**

```bash
git add evals/
git status --short | head -20
```

预期输出以 `A  evals/...` 开头，列出大量新增文件。其中许多位于 scenarios/、drill/、backends/、setup_helpers/ 等目录下。

- [ ] **Step 6: 提交**

```bash
: "${DRILL_SHA:?Set DRILL_SHA from Task 2 before committing}"
git commit -m "$(cat <<EOF
Lift drill into evals/ at $DRILL_SHA

rsync of obra/drill@$DRILL_SHA into superpowers/evals/, excluding
.git/, .venv/, results/, .env/, __pycache__/, *.egg-info/,
.private-journal/.

The drill repo is unaffected by this commit; archival is a separate
manual step after this PR merges.

Source SHA recorded in this commit message for provenance.
EOF
)"
```

---

## Task 4: 用校验和验证副本

**文件：** 无（仅验证）

- [ ] **Step 1: 获取 drill 中存在但不应出现在 evals 里的文件清单（即 excludes 的内容）**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
find . \
  \( -name '.git' -prune \
  -o -name '.venv' -prune \
  -o -name 'results' -prune \
  -o -name '__pycache__' -prune \
  -o -name '*.egg-info' -prune \
  -o -name '.private-journal' -prune \
  -o -name '*.pyc' -prune \
  -o -name '.env' -prune \) \
  -o -type f -print | sort > /tmp/drill-files.txt
wc -l /tmp/drill-files.txt
```

- [ ] **Step 2: 获取 evals/ 中的文件清单**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
find evals -type f | sed 's|^evals/|./|' | sort > /tmp/evals-files.txt
wc -l /tmp/evals-files.txt
```

- [ ] **Step 3: 对比两份清单**

在去掉被排除的路径之后，两份文件清单应当完全一致。

```bash
diff /tmp/drill-files.txt /tmp/evals-files.txt
```

预期：无输出。

- [ ] **Step 4: 逐文件校验和验证**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/drill
while read -r f; do
  sha1=$(shasum -a 256 "$f" | cut -d' ' -f1)
  sha2=$(shasum -a 256 "/Users/jesse/Documents/GitHub/superpowers/superpowers/evals/${f#./}" | cut -d' ' -f1)
  if [ "$sha1" != "$sha2" ]; then
    echo "MISMATCH: $f ($sha1 vs $sha2)"
  fi
done < /tmp/drill-files.txt | head -20
```

预期：无输出（drill 和 evals 中每个文件的校验和都匹配）。

- [ ] **Step 5: 冒烟检查——安装依赖**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv sync
```

预期：`Installed N packages` 或类似消息。无报错。

- [ ] **Step 6: 冒烟检查——drill list**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run drill list 2>&1 | head -5
```

预期：以场景名开头。（很可能报错或警告缺失 SUPERPOWERS_ROOT——没关系，下个任务修复。）

- [ ] **Step 7: 派发验证子代理**

派发一个 `general-purpose` 子代理，提示词如下：

```
You are verifying a verbatim copy of the drill repo at
/Users/jesse/Documents/GitHub/superpowers/drill into
/Users/jesse/Documents/GitHub/superpowers/superpowers/evals.

Verify:

1. The lift commit message records the SHA reported by:
  cd /Users/jesse/Documents/GitHub/superpowers/drill && git rev-parse HEAD

2. None of these excluded paths exist under evals/: .git/, .venv/,
results/, .env/, __pycache__/, *.egg-info/, .private-journal/.

3. Every non-excluded file in drill has a SHA-256-identical
counterpart in evals/, and there are no extra files in evals/.

4. The pyproject.toml, uv.lock, scenarios/*.yaml, backends/*.yaml,
setup_helpers/*.py, drill/*.py, prompts/*.md, fixtures/, bin/, and
docs/ are all present.

Report each check with PASS/FAIL. If any FAIL, dump enough detail
that the parent can fix.
```

如果子代理报告任何 FAIL，先修复底层问题（删除泄漏的文件、重新 rsync 等）再继续。

---

## Task 5: 添加 `SUPERPOWERS_ROOT` 默认值助手

**文件：**
- 修改：`evals/drill/cli.py:11-14`

- [ ] **Step 1: 读当前 cli.py 的开头**

```bash
sed -n '1,20p' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/drill/cli.py
```

预期输出：

```python
"""Drill CLI: run, compare, list."""

from __future__ import annotations

import secrets
from pathlib import Path

import click
from dotenv import load_dotenv

PROJECT_ROOT: Path = Path(__file__).parent.parent

load_dotenv(PROJECT_ROOT / ".env")
```

- [ ] **Step 2: 为该助手写一个失败的测试**

打开 `evals/tests/test_cli.py`，在末尾添加这两个测试：

```python
def test_set_superpowers_root_default_when_unset(monkeypatch, tmp_path):
    """When SUPERPOWERS_ROOT is unset, helper sets it to PROJECT_ROOT.parent."""
    monkeypatch.delenv("SUPERPOWERS_ROOT", raising=False)
    from drill.cli import _set_superpowers_root_default, PROJECT_ROOT

    _set_superpowers_root_default()

    import os
    assert os.environ["SUPERPOWERS_ROOT"] == str(PROJECT_ROOT.parent)


def test_set_superpowers_root_default_respects_existing(monkeypatch):
    """When SUPERPOWERS_ROOT is already set, helper does not override."""
    monkeypatch.setenv("SUPERPOWERS_ROOT", "/custom/path")
    from drill.cli import _set_superpowers_root_default

    _set_superpowers_root_default()

    import os
    assert os.environ["SUPERPOWERS_ROOT"] == "/custom/path"
```

- [ ] **Step 3: 运行测试，观察它失败**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_cli.py -k set_superpowers_root_default -v
```

预期：2 个测试失败，报错 `AttributeError: module 'drill.cli' has no attribute '_set_superpowers_root_default'`。

- [ ] **Step 4: 把助手加进 cli.py**

编辑 `/Users/jesse/Documents/GitHub/superpowers/superpowers/evals/drill/cli.py`。把第 1–14 行替换为：

```python
"""Drill CLI: run, compare, list."""

from __future__ import annotations

import os
import secrets
from pathlib import Path

import click
from dotenv import load_dotenv

PROJECT_ROOT: Path = Path(__file__).parent.parent

load_dotenv(PROJECT_ROOT / ".env")


def _set_superpowers_root_default() -> None:
    """Default SUPERPOWERS_ROOT to the parent of evals/ if not already set.

    Drill historically required contributors to export SUPERPOWERS_ROOT
    pointing at the superpowers checkout. After lifting drill into
    superpowers/evals/, the parent of PROJECT_ROOT is always the
    superpowers root, so we can supply this default automatically.

    Existing SUPERPOWERS_ROOT environment values are respected as overrides.
    """
    os.environ.setdefault("SUPERPOWERS_ROOT", str(PROJECT_ROOT.parent))


_set_superpowers_root_default()
```

模块底部对 `_set_superpowers_root_default()` 的调用会在 import 时执行，紧跟在 `load_dotenv()` 之后。这保证 `engine.py` 和 `setup.py`（都直接读取 `os.environ["SUPERPOWERS_ROOT"]`），以及 YAML 插值（在 backend YAML 被加载时读取 `os.environ`）都能拿到该值。

- [ ] **Step 5: 运行测试，观察它通过**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_cli.py -k set_superpowers_root_default -v
```

预期：2 个测试通过。

- [ ] **Step 6: 提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/drill/cli.py evals/tests/test_cli.py
git commit -m "evals: default SUPERPOWERS_ROOT to parent of evals/ if unset

Adds _set_superpowers_root_default() to drill/cli.py, called at
module import after load_dotenv(). PROJECT_ROOT resolves to evals/
post-lift; its parent is the superpowers repo root, which is the
correct value for SUPERPOWERS_ROOT.

Existing env values are respected as overrides via os.environ.setdefault.

Tests:
- helper sets default when var is unset
- helper does not override when var is already set"
```

---

## Task 6: 更新 backend YAML 以反映新的环境变量契约

**文件：**
- 修改：`evals/backends/codex.yaml`（从 `required_env` 中移除 `SUPERPOWERS_ROOT`）
- 修改：`evals/backends/gemini.yaml`（从 `required_env` 中移除 `SUPERPOWERS_ROOT`）

五个 `claude*.yaml` backend 配置通过 `${SUPERPOWERS_ROOT}` 插值到 `args` 里给 `--plugin-dir` flag 用——它们保留 `SUPERPOWERS_ROOT` 在 `required_env` 中，因为插值需要它。codex/gemini 配置当初列出它只是为了 engine.py/setup.py 读取 `os.environ`，这现在由助手满足。

- [ ] **Step 1: 确认当前状态**

```bash
grep -A3 'required_env:' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/codex.yaml
grep -A2 'required_env:' /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/gemini.yaml
```

预期输出包含 `- SUPERPOWERS_ROOT` 行。

- [ ] **Step 2: 完整读取 codex.yaml**

```bash
cat /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/backends/codex.yaml
```

- [ ] **Step 3: 编辑 codex.yaml——删除 `required_env` 下的 `- SUPERPOWERS_ROOT` 行**

打开 `evals/backends/codex.yaml`，找到：

```yaml
required_env:
  - OPENAI_API_KEY
  - SUPERPOWERS_ROOT
```

替换为：

```yaml
required_env:
  - OPENAI_API_KEY
```

- [ ] **Step 4: 编辑 gemini.yaml——删除 `required_env` 下的 `- SUPERPOWERS_ROOT` 行**

打开 `evals/backends/gemini.yaml`，找到：

```yaml
required_env:
  - SUPERPOWERS_ROOT
```

替换为：

```yaml
required_env: []
```

（用空列表而不是删除该字段，避免 YAML schema 校验报错。）

- [ ] **Step 5: 跑 drill 的 pytest 套件，确保没有东西被破坏**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest -x 2>&1 | tail -20
```

预期：所有测试通过。如果 `tests/test_backend.py` 对 codex/gemini 的 `required_env` 成员关系报错，见 Task 7。

- [ ] **Step 6: 提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/backends/codex.yaml evals/backends/gemini.yaml
git commit -m "evals: drop SUPERPOWERS_ROOT from codex/gemini required_env

These backends only read SUPERPOWERS_ROOT via engine.py/setup.py's
os.environ access, which the new cli.py default helper supplies
automatically. claude*.yaml keep SUPERPOWERS_ROOT in required_env
because they interpolate \${SUPERPOWERS_ROOT} into --plugin-dir args."
```

---

## Task 7: 为新契约更新 drill 的 pytest 套件

**文件：**
- 修改：`evals/tests/test_backend.py`（如果 Task 6 step 5 暴露了失败，则逐测试更新）

- [ ] **Step 1: 跑测试套件**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest tests/test_backend.py -v 2>&1 | tail -30
```

如果所有测试通过，跳到 step 5（不提交任何东西，直接进入 Task 8）。否则：

- [ ] **Step 2: 读失败的测试**

对每个失败，在 `evals/tests/test_backend.py` 中打开该测试，读其断言。

- [ ] **Step 3: 更新断言**

对于断言 `SUPERPOWERS_ROOT` 在 `codex.yaml` 或 `gemini.yaml` 的 `required_env` 中的测试：把断言反转，改为确认它不存在。示例：

```python
# Before:
def test_codex_requires_superpowers_root():
    backend = load_backend("codex")
    assert "SUPERPOWERS_ROOT" in backend.required_env

# After:
def test_codex_does_not_require_superpowers_root():
    """codex.yaml dropped SUPERPOWERS_ROOT from required_env;
    the cli.py helper supplies the default."""
    backend = load_backend("codex")
    assert "SUPERPOWERS_ROOT" not in backend.required_env
```

- [ ] **Step 4: 重新跑测试套件**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
uv run pytest -x 2>&1 | tail -10
```

预期：所有测试通过。

- [ ] **Step 5: 提交（仅当 step 1 有失败时）**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/tests/test_backend.py
git commit -m "evals: update test_backend.py for relaxed required_env contract"
```

---

## Task 8: 更新 evals/README.md 和 evals/CLAUDE.md

**文件：**
- 修改：`evals/README.md`（移除 SUPERPOWERS_ROOT 设置步骤）
- 修改：`evals/CLAUDE.md`（移除 SUPERPOWERS_ROOT 设置步骤）

- [ ] **Step 1: 编辑 evals/README.md**

找到看起来像这样的小节：

```markdown
Required environment:
```bash
export SUPERPOWERS_ROOT=/path/to/superpowers
export ANTHROPIC_API_KEY=sk-...
```
```

替换为：

```markdown
Required environment:
```bash
export ANTHROPIC_API_KEY=sk-...
```

`SUPERPOWERS_ROOT` defaults to the parent of `evals/` (the superpowers repo root) and only needs to be set if you're running drill against a different superpowers checkout.
```

- [ ] **Step 2: 编辑 evals/CLAUDE.md**

找到该小节：

```markdown
## Required env

```
SUPERPOWERS_ROOT=/path/to/superpowers
ANTHROPIC_API_KEY=sk-...
```
```

替换为：

```markdown
## Required env

```
ANTHROPIC_API_KEY=sk-...
```

`SUPERPOWERS_ROOT` defaults to the parent of `evals/` (the superpowers repo root). Override only if running drill against a different superpowers checkout.
```

- [ ] **Step 3: 提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add evals/README.md evals/CLAUDE.md
git commit -m "evals: drop SUPERPOWERS_ROOT setup step from README/CLAUDE

The cli.py helper now defaults the env var. Mention as override only."
```

---

## Task 9: 从新位置验证

**文件：** 无（仅验证——除非需要修复，否则不提交）

- [ ] **Step 1: 跑 drill 的完整 pytest 套件**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run pytest 2>&1 | tail -5
```

预期：所有测试通过。`unset` 确保我们测试的是助手本身，而不是继承来的环境变量。

- [ ] **Step 2: 运行 drill list**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill list 2>&1 | head -10
```

预期：场景清单，无关于缺失 SUPERPOWERS_ROOT 的错误。

- [ ] **Step 3: source 环境变量文件**

```bash
set -a
source /Users/jesse/Documents/GitHub/prime-radiant-inc/sprout/.env
set +a
echo "ANTHROPIC_API_KEY set: ${ANTHROPIC_API_KEY:+yes}"
```

预期：`ANTHROPIC_API_KEY set: yes`。

- [ ] **Step 4: 跑一个便宜的 drill 场景**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill run triggering-test-driven-development -b claude 2>&1 | tail -3
```

预期：`claude: 1 passed, 0 failed, 0 errors`。

如果 FAIL，继续前先排查。路径默认值的改动是最可能的元凶；可以临时在助手调用后加一句 `print(os.environ["SUPERPOWERS_ROOT"])` 来确认助手真的执行了。

---

## Task 10: Bash 测试删除阶段——逐文件用子代理把关

本任务子步骤很多，因为每个候选删除文件都有自己的子代理验证 + 提交。候选清单来自规格的覆盖映射。对下面每个条目：

1. 读该 bash 测试文件。
2. 读对应的 drill 场景 YAML。
3. 派发一个子代理，把两份内容连同比较提示词一起给它。
4. 子代理给出逐断言的匹配表。
5. 如果每个 bash 断言都有匹配：删除该 bash 测试，提交。
6. 如果有断言没匹配：停止，升级处理，不要删除。

**子代理提示词模板（每次删除都用）：**

```
You are gating a bash test deletion. The bash test is allegedly
covered by a drill scenario; your job is to verify that claim.

BASH TEST: <paste full contents of bash test>

DRILL SCENARIO: <paste full contents of drill scenario YAML>

Output a markdown table with columns: BASH ASSERTION, DRILL CHECK,
STATUS. List EVERY assertion the bash test makes (every grep, every
[ ], every test command, every PASS/FAIL emit). For each, find a
matching drill check (in verify.assertions or verify.criteria) or
mark as UNMATCHED.

After the table, output "VERDICT: SAFE TO DELETE" if every bash
assertion has a match, otherwise "VERDICT: KEEP — N unmatched
assertions". Be conservative: if you are uncertain about a match,
mark as UNMATCHED.
```

### Task 10a: 技能触发类 prompt（6 个文件）

**文件：**
- 删除：`tests/skill-triggering/prompts/dispatching-parallel-agents.txt`
- 删除：`tests/skill-triggering/prompts/executing-plans.txt`
- 删除：`tests/skill-triggering/prompts/requesting-code-review.txt`
- 删除：`tests/skill-triggering/prompts/systematic-debugging.txt`
- 删除：`tests/skill-triggering/prompts/test-driven-development.txt`
- 删除：`tests/skill-triggering/prompts/writing-plans.txt`
- 保留：`tests/skill-triggering/run-test.sh`、`run-all.sh`

这些 prompt 文件是 bash runner 的输入——它们本身没有断言。断言由 runner 脚本执行。把每个 prompt 映射到它的 drill 场景：

| Prompt | Drill 场景 |
|--------|----------------|
| dispatching-parallel-agents.txt | triggering-dispatching-parallel-agents.yaml |
| executing-plans.txt | triggering-executing-plans.yaml |
| requesting-code-review.txt | triggering-requesting-code-review.yaml |
| systematic-debugging.txt | triggering-systematic-debugging.yaml |
| test-driven-development.txt | triggering-test-driven-development.yaml |
| writing-plans.txt | triggering-writing-plans.yaml |

- [ ] **Step 1: 对每个 prompt 文件派发子代理**

对 prompt `tests/skill-triggering/prompts/<name>.txt` 和场景 `evals/scenarios/triggering-<name>.yaml`，用子代理提示词模板，把两份内容都贴进去运行。子代理的任务是验证 prompt 内容与 drill 场景中 `turns[].intent` 描述的意图是否匹配。

如果全部 6 个都判定 SAFE TO DELETE，进入 step 2。如果有任何一个判定 KEEP，那一个保留，其余的仍可继续。

- [ ] **Step 2: 确认 runner 对其他无关场景仍有用**

```bash
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/skill-triggering/prompts/
```

如果在计划删除之后 prompts/ 目录为空，同时删除 `tests/skill-triggering/run-test.sh` 和 `run-all.sh`（它们已无可跑的东西）。否则保留 runner。

- [ ] **Step 3: 删除并提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/skill-triggering/prompts/dispatching-parallel-agents.txt
git rm tests/skill-triggering/prompts/executing-plans.txt
git rm tests/skill-triggering/prompts/requesting-code-review.txt
git rm tests/skill-triggering/prompts/systematic-debugging.txt
git rm tests/skill-triggering/prompts/test-driven-development.txt
git rm tests/skill-triggering/prompts/writing-plans.txt
# If runner is now orphaned:
git rm tests/skill-triggering/run-test.sh tests/skill-triggering/run-all.sh
rmdir tests/skill-triggering/prompts/ 2>/dev/null || true
rmdir tests/skill-triggering/ 2>/dev/null || true
git commit -m "tests: remove skill-triggering bash prompts (covered by drill triggering-* scenarios)

Subagent verification confirmed each prompt's intent matches its
corresponding drill scenario's turns[].intent. Drill scenarios are
canonical; bash runner has no remaining prompts to drive."
```

### Task 10b: explicit-skill-requests（选择性删除）

**文件：**
- 审查：`tests/explicit-skill-requests/` 中的 6 个文件
- 删除：仅那些被验证为 100% 被 drill 场景覆盖的
- 保留：其余的

按规格更新后的覆盖映射，这里多数都没有 drill 对应物。可能可删的：

| Bash 测试 | 候选 drill 场景 | 可能结果 |
|-----------|--------------------------|----------------|
| `run-test.sh` | n/a（runner） | 保留 |
| `run-all.sh` | n/a（runner） | 保留 |
| `run-claude-describes-sdd.sh` | `mid-conversation-skill-invocation.yaml` | 可能删除；待验证 |
| `run-haiku-test.sh` | 无（Haiku 专用） | 保留 |
| `run-multiturn-test.sh`、`run-extended-multiturn-test.sh` | 无 | 保留 |
| `prompts/please-use-brainstorming.txt`、`prompts/use-systematic-debugging.txt` | 无 | 保留 |

- [ ] **Step 1: 逐个读 .sh 文件和 prompt 确认**

```bash
for f in /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/explicit-skill-requests/*.sh /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/explicit-skill-requests/prompts/*.txt; do
  echo "=== $f ==="
  cat "$f" | head -30
done
```

- [ ] **Step 2: 只对 `run-claude-describes-sdd.sh` 派发子代理**

用上面的子代理提示词模板，传入：
- Bash 测试内容：`tests/explicit-skill-requests/run-claude-describes-sdd.sh`
- Drill 场景：`evals/scenarios/mid-conversation-skill-invocation.yaml`

- [ ] **Step 3: 根据子代理结论行动**

如果判定 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/explicit-skill-requests/run-claude-describes-sdd.sh
git commit -m "tests: remove run-claude-describes-sdd.sh (covered by drill mid-conversation-skill-invocation)

Subagent verification: every assertion matches a drill check.
Other tests in tests/explicit-skill-requests/ are preserved
(run-haiku-test.sh, run-*-multiturn-test.sh, please-use-brainstorming
and use-systematic-debugging prompts have no drill coverage)."
```

如果判定 KEEP：跳过删除，把该缺口作为未来 drill 场景编写的任务记录下来。

### Task 10c: subagent-driven-dev 真实项目测试

**文件：**
- 审查：`tests/subagent-driven-dev/go-fractals/`、`tests/subagent-driven-dev/svelte-todo/`
- 候选场景：`evals/scenarios/sdd-go-fractals.yaml`、`evals/scenarios/sdd-svelte-todo.yaml`

这些是完整的 fixture 目录，含 `design.md`、`plan.md`、`scaffold.sh`。每个 fixture 目录在 lift 时都被作为 fixture 收纳进 `evals/fixtures/` 下。

- [ ] **Step 1: 确认 drill 有对应的 fixture 对等物**

```bash
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/fixtures/sdd-go-fractals/
ls /Users/jesse/Documents/GitHub/superpowers/superpowers/evals/fixtures/sdd-svelte-todo/
```

预期：每个都包含 `design.md`、`plan.md`、`scaffold.sh`（或等价物），与 `tests/subagent-driven-dev/` 下的源一致。

- [ ] **Step 2: 对每对派发子代理**

子代理提示词：同一模板，bash "测试" 指该目录的 `scaffold.sh` 以及（若存在）任何 `*.sh` runner。Drill 场景是对应的 `sdd-*.yaml`。

- [ ] **Step 3: 根据结论行动**

对每个判定 SAFE TO DELETE 的：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm -r tests/subagent-driven-dev/go-fractals/   # or svelte-todo
git commit -m "tests: remove subagent-driven-dev/<fixture> (covered by drill sdd-<fixture>)

Subagent verification: drill scenario asserts test suite passes
post-execution. Fixture content lives at evals/fixtures/sdd-<fixture>/."
```

如果两个目录都被删除，且 `tests/subagent-driven-dev/` 变空，再 `git rm -r tests/subagent-driven-dev/`。

### Task 10d: tests/claude-code/test-document-review-system.sh

**候选场景：** `evals/scenarios/spec-reviewer-catches-planted-flaws.yaml`

- [ ] **Step 1: 派发子代理**

用子代理提示词模板，传入 bash 测试内容和 drill 场景 YAML。

- [ ] **Step 2: 根据结论行动**

如果判定 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-document-review-system.sh
git commit -m "tests: remove test-document-review-system.sh (covered by drill spec-reviewer-catches-planted-flaws)

Subagent verification: every assertion matches a drill check."
```

### Task 10e: tests/claude-code/test-requesting-code-review.sh

**候选场景：** `evals/scenarios/code-review-catches-planted-bugs.yaml`

- [ ] **Step 1: 派发子代理**

用子代理提示词模板，传入两份内容。

- [ ] **Step 2: 根据结论行动**

如果判定 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-requesting-code-review.sh
git commit -m "tests: remove test-requesting-code-review.sh (covered by drill code-review-catches-planted-bugs)

Subagent verification: every assertion matches a drill check."
```

### Task 10f: tests/claude-code/test-worktree-native-preference.sh

**候选场景：** `evals/scenarios/worktree-creation-under-pressure.yaml`

- [ ] **Step 1: 派发子代理**

用子代理提示词模板，传入两份内容。

- [ ] **Step 2: 根据结论行动**

如果判定 SAFE TO DELETE：

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git rm tests/claude-code/test-worktree-native-preference.sh
git commit -m "tests: remove test-worktree-native-preference.sh (covered by drill worktree-creation-under-pressure)

Subagent verification: every assertion matches a drill check."
```

### Task 10g: tests/claude-code/test-subagent-driven-development-integration.sh

**候选场景：** `evals/scenarios/sdd-rejects-extra-features.yaml`（部分覆盖）

规格把这一项标为"几乎肯定保留 + 扩展 drill 场景"。不要删除。改为：

- [ ] **Step 1: 仍然派发子代理做对比**

这样能把缺口显式记录下来。

- [ ] **Step 2: 根据子代理输出决定**

可能结果：保留并记录缺口。该 bash 测试断言：`commit_count >= 3`、`npm test` 通过、运行 `analyze-token-usage.py`。而 drill 场景断言 forbidden-exports + reviewer-as-gate。两者基本不相交。

- [ ] **Step 3: 记录缺口**（如果保留）

在 `tests/claude-code/test-subagent-driven-development-integration.sh` 顶部加一条注释：

```bash
# Drill coverage: sdd-rejects-extra-features.yaml covers the YAGNI
# enforcement (forbidden exports + reviewer-as-gate). This bash test
# additionally asserts: ≥3 task commits, npm test passes, token
# analysis runs. Keep until those assertions are added to drill or
# explicitly retired.
```

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add tests/claude-code/test-subagent-driven-development-integration.sh
git commit -m "tests: annotate SDD integration test with drill coverage notes

Drill scenario sdd-rejects-extra-features covers the YAGNI subset.
This bash test adds: ≥3 commits, npm test, token analysis. Kept
until drill scenario covers those or they're retired."
```

### Task 10h: tests/claude-code/test-subagent-driven-development.sh

这是一个 meta/describe-skill 测试（按规格）。没有 drill 场景覆盖 describe-skill 行为。

- [ ] **Step 1: 通过读文件确认**

```bash
cat /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/claude-code/test-subagent-driven-development.sh
```

预期：测试是让代理描述 SDD 技能，而非演练它们。

- [ ] **Step 2: 保留并加注释**

在顶部加：

```bash
# No drill coverage: this test asks the agent to *describe* SDD
# (asserts that asked-about skills can be summarized correctly).
# Drill scenarios test behavior, not description. Kept.
```

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add tests/claude-code/test-subagent-driven-development.sh
git commit -m "tests: annotate SDD describe-skill test with kept-by-design note

Tests agent's ability to *describe* the SDD skill — drill scenarios
test behavior, not description. No drill coverage; kept by design."
```

---

## Task 11: 过时引用清理

**文件：**
- 可能修改：`docs/testing.md`、`README.md`、`CLAUDE.md`、`lefthook.yml`、`.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md`、`.github/*`、`scripts/*`
- 加注释（不重写）：`RELEASE-NOTES.md`、`docs/superpowers/plans/*.md`

- [ ] **Step 1: 构建被删除文件路径清单**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git diff --name-only --diff-filter=D dev..HEAD | sort > /tmp/deleted-paths.txt
cat /tmp/deleted-paths.txt
```

- [ ] **Step 2: 搜索活跃引用**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
while read -r path; do
  echo "=== $path ==="
  grep -rln "$path" \
    --include="*.md" \
    --include="*.yml" \
    --include="*.yaml" \
    --include="*.sh" \
    --include="*.json" \
    --exclude-dir=node_modules \
    --exclude-dir=.venv \
    --exclude-dir=evals \
    --exclude-dir=.git \
    .
done < /tmp/deleted-paths.txt
```

这会找出每一个对已删除文件的引用。对每条命中分类：

| 命中位置 | 处理方式 |
|--------------|-----------|
| `docs/testing.md` | 更新——该文档在主动记录测试 |
| `README.md`（Contributing 章节） | 如果指向已删除的测试则更新 |
| `CLAUDE.md`、`GEMINI.md`、`AGENTS.md` | 如果引用了已删除的测试则更新 |
| `.github/workflows/*.yml` | 更新——CI 不应再尝试运行已删除的测试 |
| `scripts/*` | 如果运行已删除的测试则更新 |
| `.opencode/INSTALL.md`、`.codex-plugin/INSTALL.md` | 如果引用了已删除的测试则更新 |
| `lefthook.yml` | 如果 hooks 调用了已删除的测试则更新 |
| `RELEASE-NOTES.md` | 加注释，不重写（带时间戳的历史产物） |
| `docs/superpowers/plans/*.md` | 加注释，不重写（带时间戳的历史产物） |

- [ ] **Step 3: 更新活跃引用**

对每条标为"更新"的命中，编辑文件，二选一：
- 如果被删除测试是该文件被点名的唯一原因，则移除该引用。
- 替换为指向对应 drill 场景的指针（例如 "see `evals/scenarios/triggering-test-driven-development.yaml`"）。

- [ ] **Step 4: 为带时间戳的产物加注释**

对 `RELEASE-NOTES.md` 或 `docs/superpowers/plans/*.md` 的每处命中，在每个文件的*第一处*命中位置加一段行内注释：

```markdown
> Note: this section references `tests/skill-triggering/run-all.sh` and
> related bash tests that were lifted into drill scenarios on 2026-05-06
> (see `evals/scenarios/triggering-*.yaml`). The references are
> preserved as dated artifacts of the work this doc describes.
```

不要修改实际引用——它们是历史性的。

- [ ] **Step 5: 派发子代理做第二轮清理扫描**

派发一个 `general-purpose` 子代理：

```
Working directory: /Users/jesse/Documents/GitHub/superpowers/superpowers

These bash test paths were deleted on the current branch; some are
already addressed, but I want a second pair of eyes:

<paste contents of /tmp/deleted-paths.txt>

Search the entire superpowers tree (excluding evals/, node_modules/,
.venv/, .git/) for any remaining references to those paths. Report
every hit with file:line and one-sentence judgment of whether it
needs an update or is fine as-is. Do not modify files; just report.
```

在继续之前，处理每一条被报告的命中。

- [ ] **Step 6: 提交活跃更新**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add -u  # picks up edits to existing files
git commit -m "docs: update references to lifted-and-deleted bash tests

Active references in docs/testing.md, README.md, CI workflows, etc.
now point at drill scenarios. Historical references in RELEASE-NOTES.md
and docs/superpowers/plans/*.md are annotated as dated artifacts,
not rewritten."
```

---

## Task 12: 顶层文档

**文件：**
- 修改：`docs/testing.md`——拆分为"Plugin tests"和"Skill behavior evals"
- 修改：`CLAUDE.md`——添加 evals 指针
- 修改：`README.md`——在 Contributing 章节添加指针
- 修改：`.gitignore`——添加 `evals/results/`、`evals/.venv/`、`evals/.env`

- [ ] **Step 1: 拆分 docs/testing.md**

该文件目前以 Claude Code 为中心。把它拆成两个顶层小节。

打开 `/Users/jesse/Documents/GitHub/superpowers/superpowers/docs/testing.md`，用下面的结构替换文件内容（在适用处保留现有的 Plugin-test 细节）：

```markdown
# Testing Superpowers

Superpowers has two distinct kinds of tests, each in its own directory:

- **`tests/`** — does the plugin's non-LLM code work? Bash + node + python integration tests for brainstorm-server JS, OpenCode plugin loading, codex-plugin sync, and analysis utilities.
- **`evals/`** — do agents behave correctly on real LLM sessions? Python harness driving real tmux sessions of Claude Code / Codex / Gemini CLI / Copilot CLI, with an LLM actor and verifier judging skill compliance.

## Plugin tests

Live in `tests/`. Currently:

- `tests/brainstorm-server/` — node test suite for the brainstorm server JS code.
- `tests/opencode/` — bash tests for OpenCode plugin loading, bootstrap caching, and tool registration.
- `tests/codex-plugin-sync/` — bash sync verification.
- `tests/claude-code/test-helpers.sh`, `analyze-token-usage.py` — utilities used by remaining bash tests.
- `tests/claude-code/test-subagent-driven-development.sh` — agent-can-describe-SDD test (no drill counterpart).
- `tests/claude-code/test-subagent-driven-development-integration.sh` — extended SDD integration with token analysis (drill covers the YAGNI subset).
- `tests/explicit-skill-requests/` — Haiku-specific, multi-turn, and skill-name-prompted tests not covered by drill.

Run plugin tests via the relevant directory's `run-*.sh` or `npm test`.

## Skill behavior evals

Live in `evals/`. Drill is the harness; scenarios live at `evals/scenarios/*.yaml`. See `evals/README.md` for setup. Quick start:

```bash
cd evals
uv sync
export ANTHROPIC_API_KEY=sk-...
uv run drill run triggering-test-driven-development -b claude
```

Drill scenarios are slow (3-30+ minutes each) and run real LLM sessions. They are not part of CI today; the natural follow-up is a tiered model (fast subset on PR, full sweep nightly + on-demand).
```

- [ ] **Step 2: 更新 CLAUDE.md**

读当前 CLAUDE.md，在项目结构小节附近找一个位置，加入：

```markdown
## Eval harness

Skill-behavior evals live at `evals/` — see `evals/README.md`. Drill (the harness) drives real tmux sessions of Claude Code / Codex / Gemini CLI / Copilot CLI and judges skill compliance with an LLM verifier. Plugin-infrastructure tests still live at `tests/`.
```

- [ ] **Step 3: 更新 README.md**

找到 Contributing 小节。加一行：

```markdown
- Skill-behavior tests use the eval harness at `evals/`. See `evals/README.md` for setup. Plugin-infrastructure tests live at `tests/` and run via the relevant `run-*.sh` or `npm test`.
```

- [ ] **Step 4: 更新顶层 .gitignore**

打开 `/Users/jesse/Documents/GitHub/superpowers/superpowers/.gitignore`，在底部添加：

```
# Eval harness — drill ships its own gitignore at evals/.gitignore;
# these are belt-and-suspenders entries for tools that don't recurse.
evals/results/
evals/.venv/
evals/.env
```

- [ ] **Step 5: 提交**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git add docs/testing.md CLAUDE.md README.md .gitignore
git commit -m "docs: introduce evals/ as the canonical skill-behavior eval harness

- docs/testing.md split into Plugin tests + Skill behavior evals
- CLAUDE.md adds Eval harness section pointing at evals/
- README.md Contributing section mentions evals/ alongside tests/
- .gitignore adds evals/{results,.venv,.env} as belt-and-suspenders
  (evals/.gitignore covers these locally; root-level entries help
  tooling that does not recurse into nested ignore files)."
```

---

## Task 13: 重新跑冒烟检查（回归把关）

**文件：** 无（仅验证）

- [ ] **Step 1: 跑 drill 的 pytest**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run pytest 2>&1 | tail -5
```

预期：所有测试通过。

- [ ] **Step 2: 跑便宜的 drill 场景**

```bash
set -a
source /Users/jesse/Documents/GitHub/prime-radiant-inc/sprout/.env
set +a
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/evals
unset SUPERPOWERS_ROOT
uv run drill run triggering-test-driven-development -b claude 2>&1 | tail -3
```

预期：`claude: 1 passed, 0 failed, 0 errors`。如果 FAIL，说明文档 / 清理 / 删除阶段破坏了某些东西——对最近的 commit 做 bisect。

- [ ] **Step 3: 跑保留下来的其余 plugin 测试**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers/tests/brainstorm-server
node server.test.js 2>&1 | tail -3
```

预期：`Results: 25 passed, 0 failed`。

---

## Task 14: 最终对抗式审查

**文件：** 无（仅审查；派发子代理）

- [ ] **Step 1: 为审查者构建 diff**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git log --oneline dev..HEAD
git diff dev..HEAD --stat
```

把两份输出都捕获下来，分享给审查者。

- [ ] **Step 2: 派发两个并行的子代理**

使用 `Agent` 工具，发起两次并行调用。两者用相同的提示词，以对抗式措辞：

```
Adversarial review competition: 5 points to whoever finds the most
legitimate issues. You're competing against a parallel reviewer
assigned the identical task.

**Branch:** f/evals-lift, in /Users/jesse/Documents/GitHub/superpowers/superpowers
**Base:** dev (currently b4363df)
**Spec:** docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md

This branch lifts the obra/drill repo into superpowers/evals/ and
deletes redundant bash tests that drill scenarios cover. Two prior
adversarial reviews caught issues at the spec stage; this is the
post-implementation review.

Run: git log --oneline dev..HEAD; git diff dev..HEAD --stat

Look hard at:
1. Did the rsync-with-excludes actually exclude what it claimed?
   (find evals -name '.git' -type d should return nothing)
2. Does the lift commit message point at a real commit in obra/drill?
3. Does the SUPERPOWERS_ROOT helper actually default correctly when
   the env var is unset? (cd evals && unset SUPERPOWERS_ROOT && uv
   run drill list — does it work?)
4. For each deleted bash test, does the corresponding drill scenario
   actually verify what the bash test asserted? Spot-check by reading
   the scenario YAML.
5. Are there active references in docs/, .github/, scripts/,
   lefthook.yml that still point at deleted bash test paths?
6. Did the drill pytest suite get updated for the new env-var contract,
   and does it pass?
7. Did the smoke scenario actually get run after path changes?
8. Is the drill repo unchanged? (cd ../drill && git status)

Verify before claiming. If you assert "X is broken", check on disk
first. Confidently-wrong claims count negatively.

Report format: numbered list, each with severity (critical/important/
minor/nitpick) and one-sentence explanation with file:line. Lead with
most serious. Cap at ~600 words.
```

- [ ] **Step 3: 处理发现**

对来自任一审查者的每条合理发现，在单独的 commit 中修复。修复后重新跑冒烟检查（Task 13）。

- [ ] **Step 4: 宣布胜者**

按跨平台 PR 的惯例，统计合理发现的数量（误报计负分）。在你的回复摘要中点出胜者。

---

## Task 15: 推送并开 PR

**文件：** 无

- [ ] **Step 1: 推送分支**

```bash
cd /Users/jesse/Documents/GitHub/superpowers/superpowers
git push -u origin f/evals-lift
```

- [ ] **Step 2: 用完整描述对 dev 开 PR**

```bash
gh pr create \
  --base dev \
  --head f/evals-lift \
  --reviewer arittr \
  --title "Lift drill into superpowers as evals/ harness" \
  --body "$(cat <<'EOF'
## What problem are you trying to solve?

Drill — the standalone Python skill-compliance benchmark at obra/drill — is already the de facto eval harness for superpowers. The PRI-1397 commit series lifted ~22 bash tests into drill scenarios, and the most recent superpowers commit (a2292c5) explicitly removed a redundant bash test with the message "replaced by drill behavioral coverage". Drill is a sibling repo today, requiring contributors to clone two checkouts and set SUPERPOWERS_ROOT manually. This PR completes the migration: drill becomes superpowers/evals/.

## What does this PR change?

- Lifts the obra/drill repo into superpowers as `evals/`, with explicit rsync excludes (.git, .venv, results, .env, __pycache__, *.egg-info, .private-journal). The lift commit records the source SHA.
- Adds a `_set_superpowers_root_default()` helper to drill/cli.py so SUPERPOWERS_ROOT defaults to the parent of evals/ — no manual env-var setup.
- Drops SUPERPOWERS_ROOT from required_env in codex.yaml/gemini.yaml (the helper supplies it). Claude*.yaml keep it because they interpolate ${SUPERPOWERS_ROOT} into --plugin-dir args.
- Deletes redundant bash tests under tests/skill-triggering/, tests/explicit-skill-requests/, tests/subagent-driven-dev/, and tests/claude-code/ — gated per-file by a subagent that compared each bash test's assertions to its drill scenario's verify block. Anything not 100% covered was kept.
- docs/testing.md split into Plugin tests + Skill behavior evals.
- README.md Contributing and CLAUDE.md gain pointers to evals/.

## Is this change appropriate for the core library?

Yes. Cross-runtime evaluation is core to superpowers, the migration to drill scenarios was already underway in this repo, and the eval harness needs to be discoverable in-tree to be findable.

## What alternatives did you consider?

- Vendored copy + sync script (drill repo continues independently). Rejected: divergence risk; single-source-of-truth wins.
- git subtree merge (preserves drill history in-tree). Rejected: superpowers' git history grows by 50+ commits, the merge commit is ugly, subtrees are operationally heavy.
- Keep drill as a sibling repo and just polish docs. Rejected: doesn't solve the discoverability problem.

## Does this PR contain multiple unrelated changes?

No — every change supports "drill is now evals/ inside superpowers". Multiple commits for atomicity (verbatim copy, env helper, YAML updates, docs) but one direction.

## Existing PRs

- [x] I have reviewed all open AND closed PRs for duplicates or prior art
- Related PRs: #1486 (obra/superpowers cross-platform PR — independent; no shared file changes besides README, which has no overlap)

## Environment tested

| Harness | Version | Model | Model ID |
|---------|---------|-------|----------|
| Claude Code | local install | Opus | claude-opus-4-7 (1M context) |

Drill's own pytest suite passes from the new location. `triggering-test-driven-development` drill scenario passes from `evals/` after the path-default changes. (Larger drill sweep deferred to release-cadence runs per the spec's deferred-CI policy.)

## Evaluation

- Initial prompt: see linked spec (`docs/superpowers/specs/2026-05-06-lift-drill-into-evals-design.md`).
- Drill's own pytest suite passes.
- One drill scenario re-run from the new location end-to-end (proves the SUPERPOWERS_ROOT default works).
- Per-deleted-file subagent verification recorded in each deletion commit's message.

## Rigor

- [x] If this is a skills change: this is not a skills change; it's a tooling/infrastructure migration. No behavior-shaping content modified.
- [x] Adversarial pressure-tested: two parallel reviewers on the spec; final adversarial pre-PR review on the implementation; spec already corrected for findings before implementation began.
- [x] Did not modify carefully-tuned content.

## Human review

- [x] A human has reviewed the COMPLETE proposed diff before submission

## Action items after merge

1. Archive obra/drill on GitHub (mark read-only, add README pointer to obra/superpowers/evals/).
2. The spec lists CI integration, scenario co-location with skills, and Python package rename as deferred work. Open issues for any of these you want tracked.
EOF
)"
```

- [ ] **Step 3: 确认 PR 已开**

```bash
gh pr view --web
```

预期：浏览器打开新 PR。截图或记下 URL 以便跟进。

---

## 验证清单（Task 15 之后运行）

- [ ] `git log --oneline dev..HEAD` 显示按顺序排列的预期 commit
- [ ] lift commit 消息记录了源 SHA
- [ ] `find evals -name '.git' -type d` 无输出
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run pytest` 通过
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run drill list` 返回场景
- [ ] `cd evals && unset SUPERPOWERS_ROOT && uv run drill run triggering-test-driven-development -b claude` 通过
- [ ] `tests/brainstorm-server/server.test.js` 仍然通过（非 LLM 测试的回归把关）
- [ ] `git diff dev..HEAD docs/superpowers/plans/2026-04-06-worktree-rototill.md docs/superpowers/plans/2026-03-23-codex-app-compatibility.md RELEASE-NOTES.md` 只显示注释，没有路径重写
- [ ] `cd ../drill && git log --oneline -1` 显示 obra/drill 与 lift commit 中记录的源 SHA 一致
- [ ] PR 正文列出合并后的归档待办项
