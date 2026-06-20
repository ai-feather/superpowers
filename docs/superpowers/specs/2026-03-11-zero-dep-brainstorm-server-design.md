# 零依赖 Brainstorm Server

用单个零依赖的 `server.js`（仅使用 Node.js 内置模块）替换 brainstorm 伴生服务器 vendor 进来的 node_modules（express、ws、chokidar — 共 714 个被追踪文件）。

## 动机

把 node_modules vendor 进 git 仓库会带来供应链风险：冻结的依赖拿不到安全补丁、714 个第三方代码文件未经审计就被提交、对 vendor 代码的修改看起来与普通提交无异。虽然实际风险较低（仅限本机的开发服务器），但消除这一风险很直接。

## 架构

单个 `server.js` 文件（约 250-300 行），使用 `http`、`crypto`、`fs`、`path`。该文件承担两种角色：

- **直接运行**（`node server.js`）：启动 HTTP/WebSocket 服务器
- **被 require**（`require('./server.js')`）：导出 WebSocket 协议函数供单元测试

### WebSocket 协议

仅实现 RFC 6455 的文本帧：

**握手：** 用 SHA-1 加 RFC 6455 魔法 GUID，从客户端的 `Sec-WebSocket-Key` 计算 `Sec-WebSocket-Accept`。返回 101 Switching Protocols。

**帧解码（客户端到服务端）：** 处理三种带掩码的长度编码：
- 小帧：payload < 126 字节
- 中帧：126-65535 字节（16 位扩展）
- 大帧：> 65535 字节（64 位扩展）

用 4 字节掩码密钥做 XOR 解掩码。返回 `{ opcode, payload, bytesConsumed }`；缓冲区不完整时返回 `null`。拒绝未带掩码的帧。

**帧编码（服务端到客户端）：** 不带掩码的帧，同样使用三种长度编码。

**处理的 opcode：** TEXT (0x01)、CLOSE (0x08)、PING (0x09)、PONG (0x0A)。未识别的 opcode 返回状态 1003（Unsupported Data）的关闭帧。

**刻意跳过：** 二进制帧、分片消息、扩展（permessage-deflate）、子协议。这些对本机客户端之间的小型 JSON 文本消息没有必要。扩展和子协议是在握手中协商的 — 不声明它们，它们就永远不会激活。

**缓冲区累积：** 每个连接维护一个缓冲区。收到 `data` 时追加，并循环调用 `decodeFrame` 直到它返回 null 或缓冲区为空。

### HTTP 服务器

三条路由：

1. **`GET /`** — 按 mtime 从屏幕目录中取最新的 `.html` 提供。区分完整文档与片段，将片段用 frame 模板包起来，注入 helper.js。返回 `text/html`。当没有任何 `.html` 文件时，提供一个硬编码的等待页（"Waiting for Claude to push a screen..."），并注入 helper.js。
2. **`GET /files/*`** — 从屏幕目录提供静态文件，用硬编码的扩展名映射（html、css、js、png、jpg、gif、svg、json）查 MIME 类型。找不到返回 404。
3. **其他一切** — 404。

WebSocket 升级通过 HTTP 服务器上的 `'upgrade'` 事件处理，与请求处理器分离。

### 配置

环境变量（均可选）：

- `BRAINSTORM_PORT` — 绑定端口（默认：随机高端口 49152-65535）
- `BRAINSTORM_HOST` — 绑定网络接口（默认：`127.0.0.1`）
- `BRAINSTORM_URL_HOST` — 启动 JSON 中 URL 的主机名（默认：host 为 `127.0.0.1` 时用 `localhost`，否则与 host 相同）
- `BRAINSTORM_DIR` — 屏幕目录路径（默认：`/tmp/brainstorm`）

### 启动序列

1. 若 `SCREEN_DIR` 不存在则创建（`mkdirSync` 递归）
2. 从 `__dirname` 加载 frame 模板和 helper.js
3. 在配置的 host/port 上启动 HTTP 服务器
4. 对 `SCREEN_DIR` 启动 `fs.watch`
5. 成功监听后，向 stdout 输出 `server-started` JSON：`{ type, port, host, url_host, url, screen_dir }`
6. 将同一份 JSON 写入 `SCREEN_DIR/.server-info`，使代理在 stdout 被隐藏（后台执行）时仍能找到连接信息

### 应用层 WebSocket 消息

当收到客户端的 TEXT 帧时：

1. 按 JSON 解析。若解析失败，输出到 stderr 并继续。
2. 以 `{ source: 'user-event', ...event }` 形式输出到 stdout。
3. 若事件包含 `choice` 属性，将 JSON 追加到 `SCREEN_DIR/.events`（每个事件一行）。

### 文件监听

`fs.watch(SCREEN_DIR)` 取代 chokidar。HTML 文件事件处理：

- 新增文件（某文件存在的 `rename` 事件）：若存在 `.events` 文件则删除（`unlinkSync`），以 JSON 形式向 stdout 输出 `screen-added`
- 文件变更（`change` 事件）：以 JSON 形式向 stdout 输出 `screen-updated`（不清除 `.events`）
- 两类事件都会：向所有已连接的 WebSocket 客户端发送 `{ type: 'reload' }`

按文件名去抖，约 100ms 超时，避免重复事件（在 macOS 与 Linux 上常见）。

### 错误处理

- 来自 WebSocket 客户端的畸形 JSON：输出到 stderr，继续
- 未处理的 opcode：以状态 1003 关闭
- 客户端断开：从广播集合移除
- `fs.watch` 错误：输出到 stderr，继续
- 无优雅关闭逻辑 — 进程生命周期由 shell 脚本通过 SIGTERM 处理

## 变更内容

| 之前 | 之后 |
|---|---|
| `index.js` + `package.json` + `package-lock.json` + 714 个 `node_modules` 文件 | `server.js`（单文件） |
| express、ws、chokidar 依赖 | 无 |
| 无静态文件服务 | `/files/*` 从屏幕目录提供 |

## 保持不变

- `helper.js` — 无改动
- `frame-template.html` — 无改动
- `start-server.sh` — 一行更新：`index.js` 改为 `server.js`
- `stop-server.sh` — 无改动
- `visual-companion.md` — 无改动
- 所有现有服务器行为与对外契约

## 平台兼容性

- `server.js` 只使用跨平台的 Node 内置模块
- `fs.watch` 在 macOS、Linux、Windows 上对单层平铺目录都可靠
- Shell 脚本需要 bash（Windows 上的 Git Bash，Claude Code 已要求）

## 测试

**单元测试**（`ws-protocol.test.js`）：通过 require `server.js` 的导出，直接测试 WebSocket 帧的编解码、握手计算以及协议边界情形。

**集成测试**（`server.test.js`）：测试完整服务器行为 — HTTP 服务、WebSocket 通信、文件监听、brainstorming 工作流。使用 `ws` npm 包作为仅测试用途的客户端依赖（不随分发给终端用户）。
