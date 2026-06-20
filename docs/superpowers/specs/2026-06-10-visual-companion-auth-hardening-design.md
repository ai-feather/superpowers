# Visual Companion 认证加固设计

**日期：** 2026-06-10
**状态：** Draft for Drew review

## 目标

修复在 PR #1720 的 brainstorming 可视化助手中发现的安全性和可靠性
缺陷，且不改变助手的核心工作流，也不引入运行时
依赖。

修复必须是测试优先，并留下清晰的自动化证据证明：

- 跨源浏览器标签页无法借助 cookie 注入助手事件
- 重启重连不依赖浏览器 cookie 行为也能工作
- bearer key 不会在 bootstrap 后留在可见 URL 中
- `/files/*` 不能提供内容目录之外的文件
- 未来的同源 vendored UI 库仍然工作

## 威胁模型

该助手为单个 brainstorming 会话提供 agent 生成的本地 UI。重要
资产有：

- 从助手提供的 screen 内容
- 会话 key
- `state/events`，agent 把它作为用户反馈读取
- 助手会话目录下的本地文件

范围内的攻击者：

- 另一个 `localhost` 端口上的恶意浏览器标签页
- 一个可以向助手发起请求、但不应能
  作为助手 UI 通过认证的浏览器页面
- 当服务器绑定到非环回接口时的直接远程客户端
- 通过 URL 历史、referrer 或已提交的本地状态的意外泄露
- content 目录的符号链接或逃逸 `/files/*` 的路径伎俩

本次修复范围之外：

- 恶意的 agent 编写的 screen HTML
- 由 companion screen 加载的恶意同源 vendored JavaScript

这个范围外边界是有意为之。Companion screen 是 agent UI
面的一部分。它们今天可能使用内联脚本，将来某天可能
使用同源 vendored 库，比如 Alpine 或 Three.js。防范
恶意 screen HTML 需要一个更大的沙箱化 iframe 架构，带
一个狭窄的消息桥；那不是本 PR 加固轮次的范围。

## 当前缺陷

自动化和有头浏览器测试在 PR 分支中发现了这些缺陷：

1. 一个跨源 localhost 页面可以打开一个通过 cookie 认证的 WebSocket，并
   在真实 companion 页面设置 cookie 之后向 `state/events`
   写入攻击者控制的选择。
2. `/files/*` 提供指向 `content/` 之外的符号链接，包括一个
   指向包含 keyed URL 的 `state/server-info` 的符号链接。
3. 会话 key 仍留在实际 screen 页面的 URL 中，因此同源
   screen JavaScript 以及意外的 referrer/历史可以看到它。
4. helper 用一个无 key 的 `ws://host` URL 重连。在有头 Chrome 中，
   在同端口/同 token 重启后，浏览器停止向
   重启后的服务器出示 cookie，因此打开的标签页一直卡在
   tombstone 上，直到手动 reload。
5. Shell lint 和生命周期测试需要清理，以便测试通过在
   Codex 中保持稳定。

## 设计

### 1. 带 key 的 Bootstrap 加载

`GET /?key=<token>` 变成一个 bootstrap 响应，而不是 screen 响应。

当 key 有效时，服务器：

1. 像今天一样设置 HttpOnly 会话 cookie
2. 返回一个小的 HTML bootstrap 页面
3. bootstrap 页面把 key 存储在标签页作用域的 `sessionStorage` 中
4. bootstrap 页面使用 `location.replace('/')` 导航到 `/`

此后，可见的 screen URL 是裸 `/`，而不是 `/?key=...`。

带有效 cookie 的 `GET /` 提供当前 screen。没有有效
cookie 的 `GET /` 仍返回友好的 403 页面。`GET /?key=<wrong>` 返回 403。

为什么用 `sessionStorage`：helper 需要一个能扛过
同端口重启、且不依赖 cookie 行为的重连凭据。由于 screen
HTML 是受信任的同源 UI，把 key 存在标签页作用域存储中
对这个威胁模型是可接受的。它实质上比把 key 留在
地址栏、历史和 referrer 面要好。

### 2. WebSocket 同源强制

WebSocket 升级必须通过两项检查：

1. 通过 query key 或 cookie 的有效会话认证
2. 如果存在 `Origin` 头，它必须匹配请求目标源

源检查应比较：

```text
Origin === "http://" + req.headers.host
```

浏览器攻击者页面示例：

```text
Origin: http://localhost:9999
Host: localhost:58088
```

即使浏览器发送了 companion cookie，这也必须被拒绝。

合法的 companion 页面示例：

```text
Origin: http://localhost:58088
Host: localhost:58088
```

当 key 或 cookie 有效时，这应当被接受。

直接的非浏览器客户端可能省略 `Origin`；它们仍然需要会话 key。

### 3. Helper 重连凭据

`helper.js` 应从 `sessionStorage` 读取标签页作用域的 key，并把它附加到
WebSocket URL：

```text
ws://<host>/?key=<stored-key>
```

如果不存在已存储的 key，helper 回退到当前仅依赖 cookie 的
`ws://<host>` 行为。这为已加载的、确实有有效 cookie 但没有
存储项的页面保留兼容性。

### 4. `/files/*` 围堵

文件服务器应继续拒绝空名称和 dotfile。它还必须
确保文件是 `CONTENT_DIR` 内的真实常规文件。

使用 realpath 围堵作为边界：

- 计算 `realContentDir = fs.realpathSync(CONTENT_DIR)`
- 计算 `realFilePath = fs.realpathSync(filePath)`
- 仅当 `realFilePath` 等于 `realContentDir` 的后代时才提供
- 对符号链接和内容目录之外的任何内容以 404 拒绝

服务器应继续使用 `path.basename`，以便嵌套路径仍然不被支持。

### 5. 减少泄露的头

添加保守的头，既不阻塞内联脚本，也不阻塞未来同源
vendored 库：

```text
Referrer-Policy: no-referrer
Cache-Control: no-store
X-Frame-Options: DENY
Content-Security-Policy: frame-ancestors 'none'
Cross-Origin-Resource-Policy: same-origin
```

在本次中不要添加限制性的 `script-src` CSP。companion 当前
注入内联 helper JavaScript，且未来的 screen 可能加载同源
vendored 库。

### 6. 将持久会话状态加入 Gitignore

把 `.superpowers/` 加入仓库根的 `.gitignore`，以便在使用 `--project-dir` 时，
持久化的 companion 状态和 `.last-token` 不会
被意外提交。

### 7. 测试稳定性与 lint

清理所触及的 start/stop 脚本中的 shell lint 警告。

更新调用 `start-server.sh --idle-timeout-minutes`
的生命周期测试，使其不会在 Codex 的 `CODEX_CI` 前台自动检测下挂起。该测试
在期望脚本返回启动 JSON 时应使用 `--background` 强制后台
模式。

## 测试策略

所有行为变更都应是 TDD：

1. 编写失败的聚焦测试
2. 运行它并确认它因预期原因失败
3. 实现最小修复
4. 重新运行聚焦测试
5. 重新运行完整的 brainstorm-server 套件

必需的聚焦回归测试：

- 带有效 key 的 `/` 返回 bootstrap，而不是 screen 内容
- bootstrap 把 key 存入 `sessionStorage` 并剥离 URL
- 仅 cookie 的 `/` 仍提供 screen 内容
- helper 为 WebSocket URL 使用 `sessionStorage` key
- 同源 cookie WebSocket 能打开
- 跨源 cookie WebSocket 被拒绝且不写任何事件
- 直接 key WebSocket 在没有 `Origin` 时仍能打开
- `content/` 下指向 `state/server-info` 的符号链接返回 404
- 安全头出现在正常 HTML、bootstrap、403 和文件响应上
- 同端口/token 重启能用存储的 key 认证重连
- shell lint 对触及的 shell 脚本通过
- 生命周期套件在 Codex 下不挂起

## 验收标准

- `cd tests/brainstorm-server && npm test` 反复运行通过且不挂起。
- 之前从另一个
  localhost 源写入 `attacker-injected` 的安全探测，现在无法打开 WebSocket 并保持 `state/events`
  不变。
- 指向 `server-info` 的符号链接探测返回 404。
- 有头或无头浏览器的带 key 加载最终落在裸 `/` URL 上，并且状态
  胶囊到达 Connected。
- 同端口/同 token 重启自动重连，无需手动 reload。
- `scripts/lint-shell.sh` 对触及的 shell 脚本通过。

## 推迟的工作

如果项目后来需要把 screen HTML 视为不受信任，请设计一个单独的
沙箱化 iframe 架构。那应该把生成的 screen 隔离在
单独的源或沙箱化的 frame 上，并仅暴露一个狭窄的 `postMessage` 桥
用于用户选择。不要把那并入本次修复。
