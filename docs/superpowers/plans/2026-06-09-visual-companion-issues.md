# 可视化 brainstorming 伴侣 — Issue 与变更目录

**日期：** 2026-06-09
**状态：** 分析 / 分诊。这些由我们自己实现；引用的社区 PR 仅作为证据和参考材料，**不是**我们打算合并的代码。

## 目的

用一个统一的地方记录所有与可视化 brainstorming 伴侣（位于 `skills/brainstorming/scripts/` 的本地服务器）相关的开放 issue 和 PR，提炼出底层问题以及我们将要做的改动。每一条都对照当前代码进行评估，而不是依据 PR 作者的描述。

## 范围决策（Jesse，2026-06-09）

- **不引入 Alpine.js。** PR #1639（通过引入 Alpine 构建产物实现交互式 mockup）已**放弃**。见 E3。
- **E1（terminal 对 HTML 的硬性门控）是一个 workshop 议题。** 我们会一起设计；此处不为其编写规格。
- **E2（存储位置，#975/#977）暂时搁置。**
- **远程服务是一等场景。** Superpowers 是通用工具；用户会从远程连接（SSH 隧道、Tailscale、`--host 0.0.0.0`）。安全修复必须保护这些用户，而不仅是 loopback 场景。**决策：使用按会话生成的密钥**，而不是 Host allowlist。Host allowlist 只能防御 loopback 浏览器的 confused-deputy；直连远程客户端只需发送预期的 `Host`，因此 allowlist 对远程暴露而言形同虚设。密钥是唯一能在 loopback、隧道和直连远程之间统一对客户端进行认证的东西，同时也能抵御 DNS rebinding。见 A1。

## 组件映射

| 文件 | 角色 |
|------|------|
| `skills/brainstorming/scripts/server.cjs` | 零依赖的 HTTP + WebSocket 服务器（手写实现 RFC 6455）。提供最新画面、监听 `content/`、将事件记录到 `state/events`。 |
| `skills/brainstorming/scripts/helper.js` | 注入到每个页面。WebSocket 客户端、点击捕获、`window.brainstorm` API。 |
| `skills/brainstorming/scripts/frame-template.html` | 包裹内容片段的框架（header、主题 CSS、状态点、指示条）。 |
| `skills/brainstorming/scripts/start-server.sh` | 启动封装脚本。会话目录、host/url-host、owner-PID 解析、平台后台化处理。 |
| `skills/brainstorming/scripts/stop-server.sh` | 按 PID 文件杀掉服务器，清理 `/tmp` 会话。 |
| `skills/brainstorming/visual-companion.md` | 代理在接受该伴侣时读取的操作指南。 |
| `skills/brainstorming/SKILL.md` | 伴侣被提议的位置，以及每个问题决策所在之处。 |

## 处置总结

| ID | 条目 | 来源 | 处置 |
|----|------|--------|-------------|
| A1 | 在 `/`、`/files/*` 和 WS 上使用按会话密钥（取代 Host allowlist） | issues #1014，PRs #1110/#1553 | **执行** — 选定方案 |
| A2 | Host allowlist；浏览器 WS Origin 检查 | PRs #1110/#1553 | Host allowlist 放弃；WS Origin 检查在认证之后保留，用于浏览器 confused-deputy 防御 |
| A3 | 对 `null` / 非对象 WS payload 崩溃 | PR #1504 | 执行 |
| A4 | `decodeFrame` 中对帧长度的边界限制 | issue #1446 | 已修复 — 验证后关闭 |
| B1 | dotfile 屏幕作为内容被服务（`._*.html`） | PR #950 | 执行 |
| B2 | `stop-server.sh` 杀掉被复用/过期的 PID | PR #1703 | 执行 |
| B3 | WS 客户端重连退避 + 状态指示 | PR #856 | 执行 |
| C1 | 空闲超时太短 / 不可配置；关闭时不关闭 WS | issue #1237（PR #1689） | 执行 |
| C2 | 服务器死亡对用户/代理不可见 | issue #1237（残留问题） | 执行 |
| D1 | 永久关闭该伴侣 | issue #892 | 搁置 — 不在 PR #1720 范围内 |
| D2 | 来自浏览器的自由文本反馈 | issue #957 | 搁置 — 不在 PR #1720 范围内 |
| D3 | 自动打开伴侣 URL | PR #759（#755） | 已在 PR #1720 中通过 `--open` 完成 |
| D4 | frame 中的浅色/深色对比辅助类 | PR #1683 | 搁置 — 不在 PR #1720 范围内 |
| E1 | 每个问题硬性门控 terminal 对 HTML | PR #1037 | **Workshop** |
| E2 | 将会话状态移出工作树 | issue #975（PR #977） | **搁置** |
| E3 | 引入 Alpine.js 用于交互式 mockup | PR #1639 | **放弃** |
| E4 | 启动/停止脚本中的 shell-lint 警告 | PR #1677 | 仅作顺手处理 |

---

## A. 服务器安全加固（`server.cjs`）

### A1 — 按会话密钥（选定方案）

**威胁模型。** 两类资产：被服务画面（`/`）和文件（`/files/*`）的机密性，以及 `state/events` 的完整性——一个带有 truthy `choice` 的 WebSocket 客户端会写入该文件（`server.cjs:243-246`），而代理在下一轮将其读取为用户的选择，即**对具有完整工具权限的活跃会话进行 prompt injection**。攻击路径：在默认的 `127.0.0.1` 绑定下，用户浏览器中的恶意页面（一个 confused-deputy——既运行攻击者 JS *又* 能访问 loopback）；在远程绑定（`--host 0.0.0.0`、tailnet/LAN）下，任何能路由到该端口的主机，直接访问，没有同源策略阻拦。当前 `handleUpgrade`（`server.cjs:176`）只检查 `Sec-WebSocket-Key`，而 `handleRequest`（`server.cjs:138`）什么都不检查——两者都完全敞开。

**为什么用密钥，而不是 Host allowlist。** Host allowlist 只能防御 loopback 浏览器 deputy。直连远程客户端只需发送预期的 `Host` 并伪造/省略 `Origin`，因此 allowlist 对我们恰恰必须保护的远程场景而言形同虚设。按会话密钥能在 loopback、SSH 隧道和直连远程之间统一认证客户端，同时也消灭 DNS rebinding（被反弹的页面既不知道密钥，也收不到 host 作用域的 cookie）。因此密钥**完全取代**了 A1/A2 中的 Host allowlist——不需要 `BRAINSTORM_ALLOWED_HOSTS`。

**设计。** 随机 token（`crypto.randomBytes(32)` 的十六进制），在 `server.cjs` 启动时生成（可通过 `BRAINSTORM_TOKEN` 覆盖，用于确定性测试）：

1. **URL 携带它**，形式为 `?key=<token>`。服务器已经在 `server-started` JSON 中构建了 `url`（`server.cjs:351`）并将其写入 `state/server-info`——在那里追加 `?key=` 意味着 `start-server.sh`（grep 并打印该 JSON）和技能（把这个 URL 交给用户）都**无需改动**。
2. **Cookie 引导。** `/` 上有效的 `?key` 会设置 `brainstorm-key-<port>=<token>; HttpOnly; SameSite=Strict; Path=/`。随后浏览器会自动将其附加到同源子资源（`/files/*`）和 WebSocket 握手上，因此代理可以写任意 URL 风格都能工作，`helper.js` 也无需改动。Cookie 名**按端口区分**，以避免 Jupyter 多服务器冲突（cookie 不按端口隔离）。`SameSite=Strict` 对 CDN/Unsplash 内容是安全的——该 cookie 是 host 作用域的，因此向 CDN 的出站请求永远不会携带它；SameSite 只管辖回到我们 origin 的请求，而那些都是同站的。
3. **认证门** = 在 `/`、`/files/*` 和 WS 升级上要求有效的 `?key` **或**有效的 cookie（使用 `crypto.timingSafeEqual` 比较）。缺失/错误的 key → 友好的 **403 HTML 页面**（"this page needs the full URL your coding agent gave you, including `?key=…`"——使用通用的 "coding agent"，而非 "Claude"，因为这也运行在 Codex/Gemini/Copilot 上）。WS 升级失败 → 销毁 socket。

查询 token 是事实来源；cookie 只是一种便利，从不承担初始认证负载。

**影响范围。** `server.cjs`（所有逻辑）。`helper.js` 可选的一行改动（作为 cookie 被阻止时的回退，把 `location.search` 中的 `?key=` 追加到 WS URL）。`start-server.sh` 无需改动。`visual-companion.md` 文档说明（URL 现在带有 `?key=`；不要去除它）。更新测试以传入 token。

### A2 — Host allowlist 放弃；保留浏览器 WS Origin 检查

被 A1 吸收。密钥用一种机制关闭了 WS 注入向量（#1014）、HTTP/WS DNS-rebinding 读取向量（PR #1553）以及跨源 WS 向量（PR #1110），并且与 allowlist 不同，它确实保护了远程绑定场景。不需要 `BRAINSTORM_ALLOWED_HOSTS`，也没有 Host allowlist。最终实现仍在会话认证之后检查浏览器 WebSocket 的 `Origin`，以防止跨源的 localhost 标签页借伴侣 cookie 搭车。

### A3 — 服务器在 `null` / 原始类型 WS payload 上崩溃

**问题。** `handleMessage`（`server.cjs:233`）执行 `JSON.parse(text)`，然后在 `server.cjs:243` 处 `if (event.choice)`。发送 4 字节文本帧 `null` 的客户端会产生 `event === null`，于是 `null.choice` 抛出异常。该抛出**未被捕获**——`handleMessage` 是从 `socket.on('data')` 处理器（`server.cjs:207`）中调用的，位于 `try/catch` 之外，而那个 try/catch 只包裹 `decodeFrame`。结果是未捕获异常并导致进程退出。任何本地客户端都能杀死服务器。

**改动。** 对访问加以防护：`if (event && event.choice)`。最小且精确——`JSON.parse` 不会产生 `undefined`，原始类型对 `.choice` 返回 `undefined` 而不会抛出，所以只有 `null` 是真实隐患。（避免更宽泛的修复——顶层 `try/catch` 或 `process.on('uncaughtException')` 会掩盖其他 bug。）

### A4 — `decodeFrame` 中对帧长度的边界限制（邻近）

由 PR #1504 引用为 #1446。当前代码**已经**对扩展帧长度做了边界限制：`MAX_FRAME_PAYLOAD_BYTES = 10MB`（`server.cjs:10`）在 `server.cjs:58-67` 处、任何 `Buffer.alloc` 之前强制执行。行动：对照当前 `dev` 验证 #1446，若已解决则关闭，而非重新实现。

---

## B. 服务器健壮性 / 正确性

### B1 — macOS resource-fork dotfile 被作为画面内容服务

**问题。** 最新画面选择器仅以 `f.endsWith('.html')` 做过滤（`server.cjs:127-128`）。在 macOS/ExFAT 上，`._screen.html` 这类 resource-fork 文件能通过该过滤，并且由于与真实文件并列写入，可能会排在最新位置——于是浏览器拿到的是二进制元数据而非 mockup。四处读取位置共享这个薄弱的过滤：`getNewestScreen`（`server.cjs:127`）、`knownFiles` 初始化（`server.cjs:279`）、`fs.watch` 处理器（`server.cjs:286`）以及 `/files/` 端点（`server.cjs:154-156`）。

**改动。** 在所有四处位置拒绝 dotfile（`!f.startsWith('.')`）。覆盖 `._*`、`.DS_Store` 等。

### B2 — `stop-server.sh` 可能杀掉被复用的 PID

**问题。** `stop-server.sh` 从 `state/server.pid` 读取 PID（`stop-server.sh:20`）并 `kill` 它（`:23`，在 `:35` 升级为 `-9`），而不确认该 PID 是否仍属于我们的服务器。在重启或 PID 回绕之后，该文件可能指向一个无关进程，然后我们就会对其 SIGKILL。

**改动。** 在发信号之前，验证所有权——该 PID 的命令是 `node` 且在运行我们的 `server.cjs`，理想情况下匹配当前会话。若无法证明所有权，则安全失败（报告 `stale_pid`，不杀进程）。保留已有的 `stopped` / `not_running` 输出用于真实情形。

### B3 — WebSocket 客户端：静默重连，陈旧的"已连接"

**问题。** `helper.js` 以固定 1s 定时器重连（`helper.js:21-23`），没有 `onerror` 处理器，关闭时从不把 `ws` 置空，也从不清理挂起的重连定时器。frame 的状态元素被硬编码为 "Connected"，圆点固定为 `var(--success)`（`frame-template.html:77,200`）。当笔记本休眠或服务器重启时，页面在死掉的 socket 上仍显示"Connected"，并将事件入队而毫无反馈。

**改动。**
- `helper.js`：指数退避（500ms → ×2 → 上限 30s，打开时重置）；`onerror` 委托给 `onclose`；关闭时 `ws = null`；重连前 `clearTimeout`。
- `frame-template.html`：用一个 `--status-color` 自定义属性驱动状态圆点，以便 JS 切换 Connected（绿色）/ Reconnecting（黄色）/ Disconnected（红色）。

---

## C. 生命周期 / 超时（issue #1237）

### C1 — 空闲超时太短、不可配置、WS 让进程保持存活

**问题。** `IDLE_TIMEOUT_MS` 硬编码为 30 分钟（`server.cjs:258`），由 60s 生命周期检查（`server.cjs:329-332`）强制执行。单个 brainstorm 问题在用户思考或离开时可能超过 30 分钟，于是服务器在会话中途死亡。另外，`shutdown()`（`server.cjs:310-321`）调用了 `server.close()` 但从不关闭 `clients`（`server.cjs:174`）中已升级的 socket，因此一个打开的浏览器连接能让 Node 进程在关闭之后仍然存活。

**改动。**
- 将默认值提高到 4 小时并使其可配置：`start-server.sh` 中的 `--idle-timeout-minutes` → 一个环境变量 → `IDLE_TIMEOUT_MS`，并针对 Node 定时器溢出做校验。
- 在启动 JSON / `state/server-info` 中暴露生效的超时值。
- 在 `shutdown()` 中，关闭 `clients` 中的每个 socket，让进程真正退出。

### C2 — 服务器死亡不可见

**问题。** 当服务器退出时，它会写入 `state/server-stopped` 并删除 `state/server-info`（`server.cjs:312-317`），技能也*被告知*要检查这些文件（`visual-companion.md:108`）——但这只是软性指引，模型常常跳过，而浏览器只显示一个通用的"无法访问"。用户需要手动诊断；代理却继续引用一个已失效的 URL。

**改动（两部分，与 C1 独立）：**
- **面向浏览器的墓碑页。** 在最后服务的 URL 上留下一些内容，说明"this companion expired — ask Claude to restart it"，而不是连接错误。需要权衡的方案：`helper.js` 在 socket 持续断开超过退避时间后渲染一个 banner（仅在页面已加载时有效），对比一种更复杂的方案——保留一个最小化的 responder 存活，用于服务墓碑页。
- **更严格的技能检查。** 收紧 `visual-companion.md` / `SKILL.md`，使"在引用 URL 或推送画面之前检查 `server-info`/`server-stopped`"成为一个必做步骤，而不仅仅是一句说明。保持轻量——可能只是一个代理总是运行的一行辅助命令。

---

## D. 功能

### D1 — 永久关闭可视化伴侣（issue #892）

**问题。** 该伴侣每次会话都作为独立消息被提议（`SKILL.md:25,151-152`）。一个从不想要它的用户每次都要付出那次往返——以及 HTML 生成——的代价。没有办法说"永远别再提议这个"。

**改动。** 在提议步骤之前，技能检查一个用户级设置，当设置了 opt-out 时完全跳过提议。

**设计选择待定。** 机制尚未确定：
- 环境变量（例如 `SUPERPOWERS_VISUAL_COMPANION=off`），技能被指示去读取——最简单，符合 issue 的诉求，放在 `.zshrc` 里。
- 一个插件设置文件（`.claude/superpowers.local.md` frontmatter）——更结构化、可按项目配置，但更重且是项目作用域。
- 来自 issue 的可靠性警告：一个独立的"no-companion"技能会在触发词上竞争，不够可靠——已被否决。

选定机制后，这就是一处小小的 `SKILL.md` 改动加上一个有文档说明的开关。

### D2 — 来自浏览器的自由文本反馈（issue #957）

**问题。** 客户端只捕获对 `[data-choice]` 的点击（`helper.js:36-62`）。想要给 mockup 加注（"wrong shade of blue"）的用户必须切换到终端，打断了视觉流程。

**改动。** 添加一个反馈 `<textarea>`，其提交通过已有的 `window.brainstorm.send` 路径（`helper.js:82-85`）发出 `{"type":"feedback","text":...,"timestamp":...}`。

**跨模块——需要服务器改动。** `handleMessage` 仅在 `event.choice` 为 truthy 时持久化事件（`server.cjs:243`）。`feedback` 事件没有 `choice`，因此今天它会被记录但**永远不会写入 `state/events`**，代理也看不到它。持久化条件必须也接受 `feedback` 事件。在 `visual-companion.md`（Browser Events Format，`:247-259`）中记录新的事件结构。决定提交触发方式（按钮 vs blur vs 两者）以及 textarea 渲染位置（frame 级 vs 每个画面按需）。

### D3 — 自动打开伴侣 URL（PR #759，issue #755）

**问题。** `start-server.sh` 只打印 URL；用户要手动打开。尤其在 WSL2 中，人们期望浏览器自动打开。

**改动。** 在解析 `server-started` JSON 之后尽力打开：Windows/WSL → `rundll32.exe url.dll,FileProtocolHandler <url>`，macOS → `open`，Linux → 仅当设置了 `DISPLAY`/`WAYLAND_DISPLAY` 时使用 `xdg-open`。吞掉失败，绝不阻塞启动，保持回显 URL。在 `visual-companion.md` 中记录。（考虑为无头/远程运行提供一个 opt-out，那些场景下弹出浏览器是错误的——与 D1 的配置机制相关联。）

### D4 — 浅色/深色对比辅助（PR #1683）

**问题。** 内容片段被包裹在感知操作系统的 frame 中（`frame-template.html`）。在深色模式下，快速 mockup 常常使用白色行内背景，同时继承低对比度的 frame 文本，使卡片/面板难以阅读。

**改动。** 添加 `.light-surface` / `.dark-surface` 辅助类，加上对常见行内浅色背景的保守回退，并在 `visual-companion.md` 的 CSS 参考中记录。纯 CSS，位于 `frame-template.html`。

---

## E. Workshop / 搁置 / 放弃

### E1 — 每个问题硬性门控 terminal 对 HTML（PR #1037）— WORKSHOP

软性指引已经存在："decide per-question"，在 `SKILL.md:156-161` 和 `visual-companion.md:5-25` 中有 browser-vs-terminal 测试。抱怨在于：模型对纯文本内容（A/B 列表、澄清问题）也渲染 HTML，浪费 token 和一轮对话。PR #1037 把这个决定包进了一个 `<HARD-GATE>`。**按 Jesse 的意见，我们会一起打磨措辞/机制**——这是塑造行为的技能内容，不在此处编写规格。

### E2 — 将会话状态移出工作树（issue #975 / PR #977）— 搁置

今天 `--project-dir` 把会话状态写入 `<project>/.superpowers/brainstorm/`（`start-server.sh:80-84`），技能告诉用户将其加入 gitignore（`visual-companion.md:58`）。诉求是提供一个默认在仓库之外（XDG）的 `--state-dir` / `SUPERPOWERS_STATE_DIR`，并把 `--project-dir` 保留为别名。**按 Jesse 暂时搁置。** 记录在此以免遗失。

### E3 — 引入 Alpine.js 用于交互式 mockup（PR #1639）— 放弃

加入一份 Alpine 构建产物，让 mockup 可以交互（标签页、折叠面板、表单）而无需手写 JS。**按 Jesse 放弃**——我们不会在伴侣运行时中引入第三方依赖。底层需求（交互式 mockup）不会通过这条路径推进。

### E4 — Shell-lint 警告（PR #1677）— 顺手处理

`start-server.sh` / `stop-server.sh` 中的 SC2034（及其同类）。琐碎；当我们已经在编辑 B2/C1/D3 中的脚本时顺手处理，而不是作为一项独立改动。

---

## 建议的实现分组

这些可以聚成几次连贯的迭代（每次都能独立针对 `tests/brainstorm-server/` 测试）：

1. **安全迭代**（进行中，分支 `brainstorm-companion-session-key`）——
   A1 按会话密钥（取代 A2）+ A3 null 崩溃防护。验证/关闭 A4。
   *最高优先级。*
2. **生命周期迭代** — C1 + C2 一起（两者都触及 `shutdown()` 与服务器死亡这条故事线）。
3. **健壮性迭代** — B1、B2、B3（相互独立，体量小）。
4. **搁置功能迭代** - D1、D2、D4 不在 PR #1720 范围内。D3 已通过 `--open` 流程交付。

E1 是一次单独的 workshop。E2/E3 在本轮范围之外。
