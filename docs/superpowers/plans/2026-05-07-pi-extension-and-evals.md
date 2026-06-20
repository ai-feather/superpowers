# Pi 扩展与评估实现计划

> **致代理工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐项实现本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 为 Superpowers 添加一流的 Pi 包支持，并将 Pi 作为 Drill 评估后端。

**架构：** Pi 包在根 `package.json` 中声明，加载现有 `skills/` 以及一个小型 Pi 扩展。该扩展在会话启动和压缩后，将 `using-superpowers` 引导程序作为 user 角色消息注入 provider context，并附带 Pi 专属的工具映射。Drill 新增 `pi` 后端、Pi 会话日志归一化逻辑及测试。

**技术栈：** Pi TypeScript 扩展 API、Node 内置测试运行器、Drill Python 评估宿主、pytest。

---

### Task 1: Pi 包清单与扩展测试

**文件：**
- 修改：`package.json`
- 创建：`tests/pi/test-pi-extension.mjs`

- [ ] **Step 1: 编写失败状态的包/扩展测试**

创建 `tests/pi/test-pi-extension.mjs`，测试中导入 `extensions/superpowers.ts`，注册假的 Pi handler，并断言：
- 根 `package.json` 的 `keywords` 包含 `pi-package`
- 根 `package.json` 含有 `pi.skills: ["./skills"]`
- 根 `package.json` 含有 `pi.extensions: ["./extensions/superpowers.ts"]`
- 该扩展注册了 `resources_discover`、`session_start`、`session_compact`、`context` 和 `agent_end`
- 启动时的 `context` 恰好注入一条 user 角色的引导消息
- `agent_end` 清除启动注入
- `session_compact` 重新启用注入
- 该扩展不注册 `session_before_compact`

- [ ] **Step 2: 运行测试并确认 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：FAIL，因为 `extensions/superpowers.ts` 不存在，且 `package.json` 缺少 `pi` 清单。

- [ ] **Step 3: 实现清单字段**

更新 `package.json`，加入 `description`、`keywords`、`pi.extensions` 和 `pi.skills`，同时保留已有的 `name`、`version`、`type` 和 `main`。

- [ ] **Step 4: 实现 `extensions/superpowers.ts`**

创建一个零运行时依赖的扩展，要求：
- 从 `import.meta.url` 定位包根目录
- 读取 `skills/using-superpowers/SKILL.md`
- 剥离 YAML frontmatter
- 追加 Pi 专属的工具映射
- 暴露 `resources_discover`，指向 skills 路径
- 在 `session_start` 和 `session_compact` 时将引导标记为待处理
- 在 `context` 中注入一条 user 角色引导消息
- 在前置的 `compactionSummary` 消息之后插入压缩后的引导
- 在 `agent_end` 时清除待处理引导

- [ ] **Step 5: 运行测试并确认 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：PASS。

### Task 2: Pi 工具映射参考

**文件：**
- 创建：`skills/using-superpowers/references/pi-tools.md`
- 修改：`tests/pi/test-pi-extension.mjs`

- [ ] **Step 1: 为 Pi 参考文档编写失败测试**

新增断言：`skills/using-superpowers/references/pi-tools.md` 存在，并记录了 `Skill`、`Task`、`TodoWrite` 以及内置工具名的映射。

- [ ] **Step 2: 运行测试并确认 RED**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：FAIL，因为 `pi-tools.md` 不存在。

- [ ] **Step 3: 添加 Pi 参考文档**

创建 `skills/using-superpowers/references/pi-tools.md`，说明 Pi 原生技能、可选的 `pi-subagents`、没有标准的 todo/tasklist 插件，以及内置小写工具。

- [ ] **Step 4: 运行测试并确认 GREEN**

运行：`node --experimental-strip-types --test tests/pi/test-pi-extension.mjs`

预期：PASS。

### Task 3: Drill Pi 后端与会话日志归一化

**文件：**
- 创建：`evals/backends/pi.yaml`
- 修改：`evals/drill/backend.py`
- 修改：`evals/drill/engine.py`
- 修改：`evals/drill/normalizer.py`
- 修改：`evals/tests/test_backend.py`
- 修改：`evals/tests/test_normalizer.py`

- [ ] **Step 1: 编写失败的后端/归一化测试**

为以下内容添加 pytest 覆盖：
- `load_backend("pi")` 返回 `family == "pi"`
- Pi 后端命令以 `pi` 开头，并包含 `-e ${SUPERPOWERS_ROOT}`
- Pi 的 `_resolve_log_dir()` 指向 `~/.pi/agent/sessions` 之下
- `filter_pi_logs_by_cwd()` 仅保留头部 `cwd` 与场景 workdir 匹配的会话文件
- `normalize_pi_logs()` 从 Pi assistant 会话条目中提取 `toolCall` 块，并将内置小写工具映射到标准名称

- [ ] **Step 2: 运行测试并确认 RED**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：FAIL，因为 Pi 后端和归一化器尚不存在。

- [ ] **Step 3: 添加 `evals/backends/pi.yaml`**

将后端配置为运行 `pi -e ${SUPERPOWERS_ROOT}`，使用宽松的 TUI 就绪检测、`/quit` 关闭，以及 Pi 会话日志位置。

- [ ] **Step 4: 实现 Pi family 支持**

更新 `Backend.family`、`Engine._resolve_log_dir`、`Engine._collect_tool_calls` 和 `normalizer.py`，加入 Pi 日志过滤与归一化。

- [ ] **Step 5: 运行测试并确认 GREEN**

运行：`uv run pytest evals/tests/test_backend.py evals/tests/test_normalizer.py -q`

预期：PASS。

### Task 4: 文档与完整验证

**文件：**
- 修改：`README.md`
- 修改：`evals/README.md`

- [ ] **Step 1: 记录 Pi 安装与评估后端**

将 Pi 添加到 README 的 quickstart/install 列表，并在 `evals/README.md` 中添加后端入口与用法。

- [ ] **Step 2: 运行验证**

运行：
```bash
node --experimental-strip-types --test tests/pi/test-pi-extension.mjs
uv run pytest evals/tests/test_backend.py evals/tests/test_setup.py evals/tests/test_normalizer.py -q
```

预期：所有测试通过。
