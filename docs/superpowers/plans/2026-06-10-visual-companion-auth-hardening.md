# Visual Companion 认证加固实现计划

> **给代理工作者：** 必需的子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 来逐任务地实现本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 加固 brainstorming visual companion 的认证和重连流程，同时保留可信同源屏幕 JavaScript 和未来 vendor 的 UI 库。

**架构：** 带 key 的根加载变成一个 bootstrap 步骤：设置 cookie、把 key 存入标签页级的 `sessionStorage`、然后跳转到不带参数的 `/` 屏幕 URL。WebSocket 要求有效的认证加上浏览器同源的 `Origin`，而 `/files/*` 使用 realpath 包含检查以防止逃逸出 content 目录。

**技术栈：** Node.js 内置模块（`http`、`fs`、`path`、`crypto`），零运行时依赖，已有的 `ws` 测试依赖，Bash 启停脚本，仓库 shell lint 脚本。

**重要：** 除非 Drew 明确要求，执行期间不要提交。本仓库的指令覆盖通用计划模板的提交节奏。

---

## 文件清单

- 修改：`skills/brainstorming/scripts/server.cjs`
  - 添加 bootstrap 响应。
  - 添加共享安全头。
  - 添加 WebSocket Origin 校验。
  - 添加 `/files/*` realpath 包含检查。
- 修改：`skills/brainstorming/scripts/helper.js`
  - 读取已存储的 session key 并将其附加到 WebSocket URL。
- 修改：`tests/brainstorm-server/auth.test.js`
  - 添加 bootstrap、header、同源 WS、跨源 WS、cookie/文件认证回归测试。
- 修改：`tests/brainstorm-server/helper.test.js`
  - 添加基于 sessionStorage 的 WS URL 的 mocked-browser 覆盖。
- 修改：`tests/brainstorm-server/server.test.js`
  - 添加针对 `/files/*` 的符号链接逃逸回归测试。
- 修改：`tests/brainstorm-server/lifecycle.test.js`
  - 让 start-server 超时标记测试强制使用后台模式。
  - 如果适合现有 lifecycle helper，添加重启重连凭证覆盖。
- 修改：`skills/brainstorming/scripts/start-server.sh`
  - 修复 shell lint。
- 修改：`skills/brainstorming/scripts/stop-server.sh`
  - 修复 shell lint。
- 修改：`.gitignore`
  - 添加 `.superpowers/`。
- 可选文档更新：`skills/brainstorming/visual-companion.md`
  - 如果代码行为变更需要面向操作者的说明，则提及 bootstrap URL 剥离和可信同源屏幕 JS。

## Task 1：Bootstrap 带 key 的根加载

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **Step 1：为 bootstrap 行为添加 RED 测试**

在 `tests/brainstorm-server/auth.test.js` 中，在已有的 valid-key root 测试之后添加测试：

```js
    await test('GET / with valid query returns bootstrap instead of screen content', async () => {
      const res = await get('/', { key: TOKEN });
      assert.strictEqual(res.status, 200);
      assert(res.body.includes('sessionStorage'), 'bootstrap should store the session key in tab storage');
      assert(res.body.includes('location.replace'), 'bootstrap should navigate to the bare root URL');
      assert(!res.body.includes('Secret screen'), 'bootstrap must not serve screen HTML at the keyed URL');
    });

    await test('GET / with valid cookie serves the screen after bootstrap', async () => {
      const res = await get('/', { cookie: `${COOKIE_NAME}=${TOKEN}` });
      assert.strictEqual(res.status, 200);
      assert(res.body.includes('Secret screen'), 'cookie-authenticated bare root should serve the screen');
      assert(!res.body.includes('sessionStorage'), 'bare screen response should not be the bootstrap page');
    });
```

如果已有 cookie 测试则保留；合并断言而不是重复同名测试。

- [ ] **Step 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：新的 bootstrap 测试失败，因为当前 `GET /?key=...` 直接返回 `Secret screen`，且不包含 bootstrap 的 `sessionStorage`/`location.replace` 代码。

- [ ] **Step 3：实现最小 bootstrap 响应**

在 `skills/brainstorming/scripts/server.cjs` 中，在页面常量附近添加一个 helper：

```js
function bootstrapPage(key) {
  const jsonKey = JSON.stringify(String(key));
  return `<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Opening Brainstorm Companion</title></head>
<body>
<script>
sessionStorage.setItem('brainstorm-session-key', ${jsonKey});
location.replace('/');
</script>
</body>
</html>`;
}
```

然后在 `handleRequest` 中，在认证和 cookie 设置之后、返回屏幕 HTML 之前，检测根路径上的有效 query key：

```js
function queryKey(url) {
  const q = url.indexOf('?');
  if (q < 0) return null;
  return new URLSearchParams(url.slice(q + 1)).get('key');
}
```

在 `handleRequest` 中使用它：

```js
  const pathname = pathnameOf(req.url);
  const keyFromQuery = queryKey(req.url);
  if (req.method === 'GET' && pathname === '/' && keyFromQuery && timingSafeEqualStr(keyFromQuery, TOKEN)) {
    res.writeHead(200, securityHeaders({ 'Content-Type': 'text/html; charset=utf-8' }));
    res.end(bootstrapPage(keyFromQuery));
    return;
  }
```

这里假定 Task 4 会引入 `securityHeaders`。如果先实现 Task 1，暂时使用：

```js
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
```

并在 Task 4 中替换掉。

- [ ] **Step 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：所有 auth 测试通过，包括新的 bootstrap 测试。

## Task 2：WebSocket Origin 强制校验

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **Step 1：为同源和跨源 WS 添加 RED 测试**

在 `tests/brainstorm-server/auth.test.js` 中，扩展 `wsConnect` 以接受 `origin` 选项：

```js
function wsConnect({ key, cookie, origin } = {}) {
  const url = `ws://localhost:${TEST_PORT}/` + (key !== undefined ? `?key=${key}` : '');
  const headers = {};
  if (cookie) headers['Cookie'] = cookie;
  if (origin) headers['Origin'] = origin;
  const ws = new WebSocket(url, Object.keys(headers).length ? { headers } : {});
  return new Promise((resolve) => {
    let settled = false;
    const done = (outcome) => { if (!settled) { settled = true; resolve({ outcome, ws }); } };
    ws.on('open', () => done('opened'));
    ws.on('error', () => done('rejected'));
    ws.on('close', () => done('rejected'));
    setTimeout(() => done('rejected'), 1500);
  });
}
```

然后添加：

```js
    await test('WS upgrade with valid cookie and same-origin Origin opens', async () => {
      const { outcome, ws } = await wsConnect({
        cookie: `${COOKIE_NAME}=${TOKEN}`,
        origin: `http://localhost:${TEST_PORT}`
      });
      ws.close();
      assert.strictEqual(outcome, 'opened');
    });

    await test('WS upgrade with valid cookie but cross-origin Origin is rejected', async () => {
      const eventsFile = path.join(TEST_DIR, 'state', 'events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      const { outcome, ws } = await wsConnect({
        cookie: `${COOKIE_NAME}=${TOKEN}`,
        origin: 'http://localhost:9999'
      });
      if (outcome === 'opened') {
        ws.send(JSON.stringify({ type: 'choice', choice: 'attacker-injected', text: 'local attacker probe' }));
        await sleep(300);
      }
      ws.close();

      assert.strictEqual(outcome, 'rejected', 'cross-origin browser WS must not open even with cookie');
      assert(!fs.existsSync(eventsFile), 'cross-origin WS must not write state/events');
    });
```

- [ ] **Step 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：跨源 cookie WS 测试失败，因为当前服务器接受任何带 cookie 认证的 WS 而不管 Origin。

- [ ] **Step 3：实现 Origin 检查**

在 `skills/brainstorming/scripts/server.cjs` 中，添加：

```js
function isAllowedWebSocketOrigin(req) {
  const origin = req.headers.origin;
  if (!origin) return true; // non-browser clients still need the session key
  const host = req.headers.host;
  if (!host) return false;
  return origin === 'http://' + host;
}
```

然后更新 `handleUpgrade`：

```js
function handleUpgrade(req, socket) {
  if (!isAuthorized(req) || !isAllowedWebSocketOrigin(req)) { socket.destroy(); return; }
```

- [ ] **Step 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：auth 测试通过；跨源 WS 被拒绝；同源和直接 key 的 WS 仍然能打开。

## Task 3：Helper 使用存储的 key 进行重连

**文件：**
- 修改：`tests/brainstorm-server/helper.test.js`
- 修改：`skills/brainstorming/scripts/helper.js`

- [ ] **Step 1：为 WebSocket URL key 添加 RED 测试**

在 `tests/brainstorm-server/helper.test.js` 中，在重连状态机测试附近添加一个 mocked-browser 测试：

```js
test('uses sessionStorage key in the WebSocket URL when present', () => {
  const e = makeEnv();
  e.state.sessionKey = 'stored-key-abc';
  e.boot();
  assert.strictEqual(e.sockets[0].url, 'ws://localhost:7777/?key=stored-key-abc');
});
```

更新 `makeEnv()`，让返回的对象暴露 `sockets`，并且 mock window 包含 sessionStorage：

```js
    window: {
      location: { host: 'localhost:7777', reload() { state.reloads++; } },
      sessionStorage: { getItem: (key) => key === 'brainstorm-session-key' ? state.sessionKey : null }
    },
```

同时添加一个回退测试：

```js
test('uses cookie-only WebSocket URL when no sessionStorage key is present', () => {
  const e = makeEnv();
  e.state.sessionKey = null;
  e.boot();
  assert.strictEqual(e.sockets[0].url, 'ws://localhost:7777');
});
```

- [ ] **Step 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node helper.test.js
```

预期：stored-key 测试失败，因为当前 helper 使用 `ws://localhost:7777`。

- [ ] **Step 3：实现存储 key 的 WS URL**

在 `skills/brainstorming/scripts/helper.js` 中，把：

```js
  const WS_URL = 'ws://' + window.location.host;
```

替换为：

```js
  function websocketUrl() {
    let key = null;
    try { key = window.sessionStorage && window.sessionStorage.getItem('brainstorm-session-key'); } catch (e) {}
    return 'ws://' + window.location.host + (key ? '/?key=' + encodeURIComponent(key) : '');
  }
```

然后把：

```js
    ws = new WebSocket(WS_URL);
```

替换为：

```js
    ws = new WebSocket(websocketUrl());
```

- [ ] **Step 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node helper.test.js
```

预期：helper 测试通过。

## Task 4：安全响应头

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **Step 1：添加 RED header 测试**

在 `tests/brainstorm-server/auth.test.js` 中，添加：

```js
    await test('HTML responses include leak-reduction and anti-framing headers', async () => {
      const res = await get('/', { key: TOKEN });
      assert.strictEqual(res.headers['referrer-policy'], 'no-referrer');
      assert.strictEqual(res.headers['cache-control'], 'no-store');
      assert.strictEqual(res.headers['x-frame-options'], 'DENY');
      assert.strictEqual(res.headers['content-security-policy'], "frame-ancestors 'none'");
      assert.strictEqual(res.headers['cross-origin-resource-policy'], 'same-origin');
    });

    await test('403 responses include leak-reduction and anti-framing headers', async () => {
      const res = await get('/');
      assert.strictEqual(res.status, 403);
      assert.strictEqual(res.headers['referrer-policy'], 'no-referrer');
      assert.strictEqual(res.headers['cache-control'], 'no-store');
      assert.strictEqual(res.headers['x-frame-options'], 'DENY');
      assert.strictEqual(res.headers['content-security-policy'], "frame-ancestors 'none'");
      assert.strictEqual(res.headers['cross-origin-resource-policy'], 'same-origin');
    });
```

- [ ] **Step 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：header 测试失败，因为当前响应不包含这些头。

- [ ] **Step 3：实现共享 header helper**

在 `skills/brainstorming/scripts/server.cjs` 中，添加：

```js
function securityHeaders(headers = {}) {
  return {
    'Referrer-Policy': 'no-referrer',
    'Cache-Control': 'no-store',
    'X-Frame-Options': 'DENY',
    'Content-Security-Policy': "frame-ancestors 'none'",
    'Cross-Origin-Resource-Policy': 'same-origin',
    ...headers
  };
}
```

更新 `handleRequest` 中的响应写入：

```js
res.writeHead(403, securityHeaders({ 'Content-Type': 'text/html; charset=utf-8' }));
```

```js
res.writeHead(200, securityHeaders({ 'Content-Type': 'text/html; charset=utf-8' }));
```

```js
res.writeHead(200, securityHeaders({ 'Content-Type': contentType }));
```

对于 404：

```js
res.writeHead(404, securityHeaders());
```

- [ ] **Step 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：auth 测试通过且 header 断言为 green。

## Task 5：`/files/*` Realpath 包含检查

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **Step 1：添加 RED 符号链接逃逸测试**

在 `tests/brainstorm-server/server.test.js` 中，在 `/files/` 空文件名测试之后添加：

```js
    await test('does not serve symlinks that escape content dir via /files/', async () => {
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'linked-server-info.txt');
      try { fs.unlinkSync(link); } catch (e) {}
      fs.symlinkSync(target, link);

      const res = await fetch(`http://localhost:${TEST_PORT}/files/linked-server-info.txt`);
      assert.strictEqual(res.status, 404, 'symlink to state/server-info must not be served');
      assert(!res.body.includes('server-started'), 'response must not include server-info body');
    });
```

- [ ] **Step 2：验证 RED**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node server.test.js
```

预期：符号链接测试失败，因为当前 `/files/*` 跟随符号链接并返回 `server-info`。

- [ ] **Step 3：实现包含检查 helper**

在 `skills/brainstorming/scripts/server.cjs` 中，添加：

```js
function isRegularFileInsideContentDir(filePath) {
  let stat, realContentDir, realFilePath;
  try {
    stat = fs.lstatSync(filePath);
    if (stat.isSymbolicLink()) return false;
    if (!stat.isFile()) return false;
    realContentDir = fs.realpathSync(CONTENT_DIR);
    realFilePath = fs.realpathSync(filePath);
  } catch (e) {
    return false;
  }
  return realFilePath.startsWith(realContentDir + path.sep);
}
```

把 `/files/*` 的守卫替换为：

```js
    if (!fileName || fileName.startsWith('.') || !isRegularFileInsideContentDir(filePath)) {
      res.writeHead(404, securityHeaders());
      res.end('Not found');
      return;
    }
```

- [ ] **Step 4：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node server.test.js
```

预期：server 测试通过，包括符号链接拒绝。

## Task 6：重启重连回归

**文件：**
- 修改：`tests/brainstorm-server/lifecycle.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`
- 修改：`skills/brainstorming/scripts/helper.js`

- [ ] **Step 1：为重启后用同一 key 通过 WS 认证添加 RED 集成测试**

在 `tests/brainstorm-server/lifecycle.test.js` 中，在 port/token 持久化测试之后添加一个测试：

```js
  await test('stored key can authenticate WebSocket after same-port restart', async () => {
    const dir = fs.mkdtempSync('/tmp/bs-reconnect-');
    const portFile = path.join(dir, '.last-port');
    const tokenFile = path.join(dir, '.last-token');
    const env = { ...process.env, BRAINSTORM_PORT_FILE: portFile, BRAINSTORM_TOKEN_FILE: tokenFile, BRAINSTORM_LIFECYCLE_CHECK_MS: 100000 };

    const a = spawn('node', [SERVER], { env: { ...env, BRAINSTORM_DIR: path.join(dir, 's1') } });
    let outA = ''; a.stdout.on('data', d => outA += d.toString());
    for (let i = 0; i < 60 && !outA.includes('server-started'); i++) await sleep(50);
    const infoA = firstServerStarted(outA);
    const keyA = new URL(infoA.url).searchParams.get('key');
    a.kill(); await sleep(400);

    const b = spawn('node', [SERVER], { env: { ...env, BRAINSTORM_DIR: path.join(dir, 's2') } });
    let outB = ''; b.stdout.on('data', d => outB += d.toString());
    for (let i = 0; i < 60 && !outB.includes('server-started'); i++) await sleep(50);
    const infoB = firstServerStarted(outB);

    const ws = new WebSocket(`ws://localhost:${infoB.port}/?key=${keyA}`, {
      headers: { Origin: `http://localhost:${infoB.port}` }
    });
    const opened = await new Promise(resolve => {
      ws.on('open', () => resolve(true));
      ws.on('error', () => resolve(false));
      setTimeout(() => resolve(false), 1500);
    });

    try {
      assert.strictEqual(infoB.port, infoA.port, 'restart should reuse same port');
      assert(opened, 'stored key should authenticate WS after restart');
    } finally {
      try { ws.close(); } catch (e) {}
      b.kill(); await sleep(100);
      fs.rmSync(dir, { recursive: true, force: true });
    }
  });
```

这个测试在 Task 2 和 Task 3 实现后可能已经通过。如果在代码改动前它就已通过，保留它作为覆盖但不要称之为 RED。真实的浏览器重连行为主要由 Task 3 加上最终的人工/headless 浏览器验证覆盖。

- [ ] **Step 2：验证行为**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node lifecycle.test.js
```

预期（在 Task 2 和 Task 3 之后）：lifecycle 测试通过。如果失败，在继续之前修复认证/重启路径。

## Task 7：Lifecycle 挂起与 Shell Lint

**文件：**
- 修改：`tests/brainstorm-server/lifecycle.test.js`
- 修改：`skills/brainstorming/scripts/start-server.sh`
- 修改：`skills/brainstorming/scripts/stop-server.sh`

- [ ] **Step 1：复现 shell lint 失败**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/stop-server.test.sh
```

预期的当前失败：

```text
SC2164: skills/brainstorming/scripts/start-server.sh line 128: cd "$SCRIPT_DIR"
SC2034: skills/brainstorming/scripts/start-server.sh line 166: for i in {1..50}
SC2034: skills/brainstorming/scripts/stop-server.sh line 57: for i in {1..20}
```

- [ ] **Step 2：最小化修复 shell lint**

在 `skills/brainstorming/scripts/start-server.sh` 中，把：

```bash
cd "$SCRIPT_DIR"
```

改为：

```bash
cd "$SCRIPT_DIR" || exit 1
```

对于未读取的循环变量，把 `i` 改为 `_`：

```bash
for _ in {1..50}; do
```

在 `skills/brainstorming/scripts/stop-server.sh` 中，把：

```bash
for i in {1..20}; do
```

改为：

```bash
for _ in {1..20}; do
```

- [ ] **Step 3：修复 lifecycle start-server 挂起**

在 `tests/brainstorm-server/lifecycle.test.js` 中，更新 `start-server.sh --idle-timeout-minutes sets the timeout` 测试命令：

```js
const out = execFileSync('bash', [START, '--project-dir', dir, '--idle-timeout-minutes', '5', '--background'], { encoding: 'utf8' });
```

这样当 `CODEX_CI` 触发 start-server 前台模式时，测试不会挂起。

- [ ] **Step 4：验证 lint 和 lifecycle**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/stop-server.test.sh
cd tests/brainstorm-server
node lifecycle.test.js
```

预期：shell lint 以 0 退出；lifecycle 测试以 0 退出且不挂起。

## Task 8：Gitignore 持久化 Companion 状态

**文件：**
- 修改：`.gitignore`

- [ ] **Step 1：验证当前忽略缺口**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
git check-ignore .superpowers/brainstorm/.last-token || true
```

预期的当前输出：没有匹配的忽略规则。

- [ ] **Step 2：添加忽略规则**

向 `.gitignore` 添加这一行：

```gitignore
.superpowers/
```

- [ ] **Step 3：验证 GREEN**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
git check-ignore .superpowers/brainstorm/.last-token
```

预期输出：

```text
.superpowers/brainstorm/.last-token
```

## Task 9：完整自动化验证

**文件：**
- 本任务无代码改动。

- [ ] **Step 1：运行聚焦测试套件**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
node auth.test.js
node helper.test.js
node server.test.js
node lifecycle.test.js
```

预期：全部四条命令以 0 退出。

- [ ] **Step 2：运行完整 brainstorm-server 套件**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
npm test
```

预期：所有测试通过，包括 ws-protocol、helper、auth、server、lifecycle 和 stop-server。

- [ ] **Step 3：针对 lifecycle/watch 抖动重复运行套件**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers/tests/brainstorm-server
for i in 1 2 3; do npm test || exit 1; done
```

预期：三次重复全部通过且不挂起。

- [ ] **Step 4：运行 shell lint**

运行：

```bash
cd /Users/drewritter/prime-rad/superpowers
scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/stop-server.test.sh
```

预期：以 0 退出。

## Task 10：重跑安全探测

**文件：**
- 本任务无代码改动。

- [ ] **Step 1：重建跨源攻击者探测**

如果可用，使用之前的临时探测：

```bash
node /tmp/superpowers-pr1720-security-drewritter/probe-pr1720.cjs
```

如果临时探测不可用，在 `/tmp` 下重建一个最小探测，要求：

- 以固定 token 启动 companion
- 在 headless Chrome 中加载带 key 的 URL
- 在不同的 localhost 端口上启动攻击者页面
- 尝试 `new WebSocket('ws://localhost:<companion-port>/')`
- 发送 `{"type":"choice","choice":"attacker-injected"}`
- 检查 `state/events`

修复后的预期：

- 无 key 和错误 key 的 HTTP 仍然返回 403
- 同源 helper 到达 Connected
- 跨源 WebSocket 不会打开
- `state/events` 不包含 `attacker-injected`
- 指向 `server-info` 的符号链接返回 404
- 带 key 的浏览器加载最终落在不带参数的 `/`

- [ ] **Step 2：仅在自动化探测通过后，重跑人工/浏览器流程**

人工流程：

1. 用 `--project-dir --open` 启动 companion
2. 推送一个屏幕
3. 确认 URL 被剥离为 `/`
4. 确认状态到达 Connected
5. 点击一个选项并验证 `state/events`
6. 停止并重启同一 project
7. 验证打开的标签页自动重连

预期：所有步骤通过，无需人工重载 URL。

## 自检清单

- 规格覆盖：每一项设计要求都映射到至少一个任务。
- 占位符扫描：本计划不包含任何未解决的占位符标记或未指定的边界情况步骤。
- TDD 顺序：每个生产改动任务都以一个聚焦的失败测试或一条能演示当前失败的命令开始。
- 信任模型：本计划保留了可信同源屏幕 JavaScript 和未来同源 vendor 库。
- 不提交规则：除非 Drew 明确要求，执行期间不提交。
