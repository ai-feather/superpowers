# Visual Companion 最终加固修复设计

**日期：** 2026-06-11
**状态：** 待 Drew 审查的草稿

## 目标

完成 PR #1720 visual companion 加固轮次，使分支具备干净的安全行为、确定性的测试、以及一个只含 companion 工作的 PR diff，准备好接受 Jesse 审查。

这是已有 auth 加固设计之上的一次修复。它不应重新设计 companion，也不应扩展功能面。

## 背景

前一轮加固加入了 keyed sessions、同源 WebSocket 检查、URL key 去除、`/files/*` 围堵、减少泄漏的响应头、IPv6 URL 格式化、Windows 生命周期覆盖，以及 PR 凭据更新。

最终审查轮发现了五个遗留问题：

1. 根 `GET /` 屏幕选择路径仍可能服务 `content/` 下的符号链接或硬链接，指向内容目录之外。
2. 当首选端口被占用时，回退服务器可能复用一个已持久化的 `.last-token`，从而产生两个使用同一 bearer key 的、同项目的活动 companion 服务器。
3. 当强所有权证明不可用时，`stop-server.sh` 可能向一个不相关的 `node server.cjs` 进程发信号。
4. 一些测试可能针对错误的回退进程通过、在失败时泄漏后台进程，或者在类 Windows 主机上假定支持符号链接。
5. 该 PR 当前处于冲突状态，因为分支包含一次较早的 `evals` submodule bump，而它已被单独处理。

## 非目标

- 本轮不添加 HTTPS 隧道或 `wss://` origin 语义。
- 不实现 opt-out、自由文本或对比辅助类的 companion 功能。
- 不 vendor Alpine、Three.js 或任何其他 JavaScript 库。
- 不尝试沙箱化恶意的、代理生成的屏幕 HTML。
- 不为陈旧的 stop-server PID 文件添加向后兼容，除非 Drew 明确批准这一权衡。

## 继承的安全不变量

本次修复保留已设计并实现的 auth 加固：

- `.last-token` 和 `state/server-info` 仍为敏感的、仅属主可写的状态。
- 回退 token 可能出现在启动 JSON 和 `state/server-info` 中，但绝不能被写入 `.last-token`。
- Cookie 仍以端口命名、`HttpOnly`、`SameSite=Strict`、限定于 `/`。
- WebSocket 升级仍要求有效 key 或 cookie。
- 当浏览器提供 `Origin` 头时，WebSocket `Origin` 检查仍被强制执行。
- 直接的、不带 `Origin` 的客户端仍只有在携带会话 key 时才被允许。
- 生成的同源屏幕 JavaScript 与未来的同源 vendored 库是受信任的。沙箱化恶意屏幕 HTML 仍被推迟。

## 设计

### 1. 在当前 `dev` 上 rebase

在实现工作之前，把 `brainstorming-companion` rebase 到当前 `origin/dev` 上。对 `evals` submodule 冲突取 `dev` 版本解决。

rebase 之后：

- `evals` 不得出现在 PR diff 中。
- PR #1720 仍可提到在别处运行的 eval 证据，但必须包含确切的外部凭据：eval 仓库提交、场景路径、命令、结果工件路径或 id，以及 RED/GREEN 结果。
- PR 正文不得暗示 evals submodule bump 是本 PR 的一部分。
- 任何早先暗示该 submodule bump 已包含的 PR 正文文本或评论，都必须被最终 PR 正文凭据取代。

### 2. 根屏幕围堵

根屏幕路由必须使用与 `/files/*` 相同的围堵边界。

`getNewestScreen()` 应忽略任何未通过"常规文件且位于内容目录之内"守卫的 `.html` 候选。该守卫必须解析真实路径并确保被服务的文件位于 `CONTENT_DIR` 内。它还必须保留现有的硬链接保护：当平台报告链接数时，拒绝链接数不为 1 的文件。

预期行为：

- `content/` 下指向 `content/` 之外的符号链接被忽略。
- 当 `fs.linkSync` 成功且 `lstat.nlink > 1` 时，`content/` 下指向 `state/server-info` 的硬链接被忽略。
- 如果没有安全的屏幕文件剩下，服务等待页。
- 现有 `/files/*` 围控行为保持不变：空名、dotfile、符号链接、硬链接和目录仍返回 404。

### 3. 回退 Token 隔离

端口回退绝不能复用从持久化 `.last-token` 加载的 token。

token 来源在代码中应当显式：

- 来自环境的 `BRAINSTORM_TOKEN` 是运维/测试的有意覆盖。如果首选端口被占用且同时设置了显式环境 token，服务器必须 fail closed 而非回退，因为被占用的服务器可能正在使用相同的显式 token。
- `.last-token` 是为同端口重连便利而持久化的状态。如果服务器因首选端口被占用而回退，丢弃该已加载 token，并为回退进程生成一个新鲜的、未持久化的 token。
- 一个不是从 `.last-token` 加载的新生成 token 可以在同一进程内复用，因为不知道有其他活动进程持有它。

回退服务器必须继续避免覆盖 `.last-port` 和 `.last-token`。

### 4. Stop-Server 所有权证明

`start-server.sh` 应创建一个每次启动唯一的 server instance id，并把它作为惰性命令行参数传给 Node，例如：

```text
node server.cjs --brainstorm-server-id=<id>
```

该 id 不是认证凭据。它只是本地生命周期脚本的进程归属证据。`server.cjs` 可以忽略该参数。

该 id 必须使用 shell/MSYS 安全的字母表，例如 `^[A-Za-z0-9_-]{32,64}$`。把它存入 `state/server-instance-id`，权限为仅属主可写。

`stop-server.sh` 应从 state 读出期望的 id，仅当目标进程 argv 包含作为完整 argv token 的精确参数 `--brainstorm-server-id=<id>`（而非松散子串）时才向 PID 发信号。优先使用 `/proc/<pid>/cmdline`（如可用），然后回退到宽输出 `ps`。即便 `server-info` 缺失或 `lsof` 不可用，匹配的 instance id 也是充分证明。现有的端口到 PID 检查可作为额外证据保留。

当无法证明所有权时 fail closed：

- 缺失 PID 文件
- 缺失或格式错误的 server id
- 目标命令行不可用
- 目标命令行不含期望 id
- 没有新 id 的旧/陈旧会话元数据

这是有意选择宁可让陈旧进程继续运行，也不要杀掉一个不相关的进程。

对运维可见的结果应当显式：

- 缺失 PID 文件返回 `not_running`
- 缺失或格式错误的 server id 返回 `stale_pid`
- 命令行不可用返回 `stale_pid`
- argv id 错误或缺失返回 `stale_pid`
- 成功停止返回 `stopped`

在 `stale_pid` 与 `stopped` 结果下，移除 `server.pid` 与 `server-instance-id`，以便未来停止尝试不再持续指向同一歧义进程。不要移除持久会话内容。

### 5. 测试加固

测试通过必须在 macOS 和用于验证的 Windows Git Bash 主机上都是确定性的。

所需变更：

- 固定端口套件必须在服务器报告回退端口时立即 fail fast，或者让所有客户端都使用报告的启动端口。
- `stop-server.test.sh` 需要在启动任何后台进程之前设置顶层清理 trap。
- 符号链接相关断言应先探测符号链接能力，当主机无法创建可用的测试符号链接时仅跳过该断言。
- 创建冒名进程的测试必须断言：当生命周期元数据缺失或不充分时，冒名进程存活。
- Windows/MSYS 的 start-server 测试必须断言：类 Windows 检测仍然清除 `BRAINSTORM_OWNER_PID`、在合适时仍然自动前台化，并且仍然精确透传 instance-id argv。

### 6. 文档与 PR 一致性

在 Jesse 审查前，对齐审查者可见的文档与 PR 元数据：

- 更新 issue 清单，使处置与本 PR 实际交付一致。
- 保持自动打开文档与已实现的 `--open` 行为一致。
- 在所有地方保持文档化的默认 idle timeout 为 4 小时。
- rebase 之后对照模板审查 PR 正文。
- 在 PR 正文中记录 macOS、Windows、浏览器/手动与外部 eval 凭据，附具体命令与结果。

## 测试策略

对每项行为变更采用 TDD：

1. 新增或收紧一个聚焦的回归测试。
2. 运行它并确认它因预期原因失败。
3. 实现最小修复。
4. 重跑聚焦测试。
5. 重跑完整 brainstorm-server 套件。

所需聚焦回归项：

| 行为 | 测试文件 | 聚焦命令 | 预期 RED | 预期 GREEN |
| --- | --- | --- | --- | --- |
| 根路由忽略符号链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已鉴权的 `GET /` 服务链接到内容之外 | 响应服务等待页或安全屏幕 |
| 根路由忽略受支持的硬链接逃逸 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 已鉴权的 `GET /` 服务硬链接的 `server-info` | 当 `nlink > 1` 时硬链接候选被忽略 |
| `/files/*` 围堵保持不变 | `tests/brainstorm-server/server.test.js` | `node tests/brainstorm-server/server.test.js` | 现有围堵测试回归 | 空、dotfile、目录、符号链接、硬链接各情形仍为 404 |
| 持久化 token 回退轮换 token | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 URL key 等于持久化的首选端口 key | 回退 URL key 不同，且不写入 `.last-token` |
| 显式 token 回退 fail closed | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 服务器在设置了 `BRAINSTORM_TOKEN` 时回退 | 进程以非零退出，且不启动回退 |
| 回退 key 无法对原服务器鉴权 | `tests/brainstorm-server/lifecycle.test.js` | `node tests/brainstorm-server/lifecycle.test.js` | 回退 key 从原端口收到 200 | 原端口拒绝回退 key |
| 正确的 instance id 允许停止 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 真实 start-server 启动的服务器存活 | 停止返回 `stopped` 且进程退出 |
| 错误、缺失、格式错误或陈旧的 id 是安全的 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 冒名进程被发信号 | 停止返回 `stale_pid` 且冒名进程存活 |
| 固定端口套件不能透过回退通过 | `tests/brainstorm-server/server.test.js`, `tests/brainstorm-server/auth.test.js` | 各自的 `node` 命令 | 测试静默地与回退端口对话 | 测试明确失败或有意使用报告的端口 |
| Shell 清理 trap 在失败时运行 | `tests/brainstorm-server/stop-server.test.sh` | `bash tests/brainstorm-server/stop-server.test.sh` | 失败留下子进程 | trap 回收后台子进程 |
| Windows/MSYS 启动行为保持生命周期不变量 | `tests/brainstorm-server/start-server.test.sh`, `tests/brainstorm-server/windows-lifecycle.test.sh` | macOS 与 `ballmer` 上的 `bash` 测试命令 | 属主 PID 或 argv 处理回归 | 属主 PID 被清除、前台检测保持、id argv 存在 |

每个 RED/GREEN 循环应在 PR 正文中留下一则简短凭据说明：聚焦命令、修复前的失败断言、修复后的通过断言，以及凭据是在 macOS 还是 Windows 上采集。

## 验证

在宣布本次修复完成之前，运行：

- `git fetch origin dev && git rebase origin/dev`
- `git diff --quiet origin/dev...HEAD -- evals`
- `gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid`
- `cd tests/brainstorm-server && npm test`
- TDD 期间用到的相关聚焦测试命令
- `git diff --check`
- 对改动过的 JavaScript 文件做 Node 语法检查
- 对改动过的 shell 文件做 shell lint
- 在 `ballmer` 上做 Windows 验证：完整可运行的 brainstorm-server 套件加独立的 Windows 生命周期探针

手动/浏览器测试只在自动化通过之后进行。

## 验收标准

- PR #1720 干净地 rebase 到当前 `dev`。
- PR diff 中不含 `evals`。
- 根屏幕服务不能通过符号链接或受支持的硬链接逃逸读到 `content/` 之外。
- `/files/*` 围堵保护保持不变。
- 没有回退服务器以可能与被占用的首选端口服务器共享的 token 运行。
- 当所有权证明缺失或歧义时，`stop-server.sh` 不向不相关进程发信号。
- 当 `server-info` 或 `lsof` 不可用时，`stop-server.sh` 仍能以匹配的 instance id 停止一个合法服务器。
- 每个回归都记录了聚焦的 RED/GREEN 凭据。
- macOS 与 Windows 验证凭据记录在 PR 正文中。
- PR 正文准确描述分支内容与在外部采集的凭据。
