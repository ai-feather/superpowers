# 可视化伴侣最终加固修复实施计划

> **致代理型工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 按任务逐项实施本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 以测试先行的改动完成 PR #1720 的最终加固修复，获得干净的 rebase 状态，以及可供审阅者查阅的证据。

**规格：** `docs/superpowers/specs/2026-06-11-visual-companion-final-hardening-fixup-design.md`

**架构：** 保持伴侣零依赖且本地优先。向现有服务器和 shell 脚本添加聚焦的防护：根屏幕选择复用 `/files/*` 包含防护，回退 token 处理跟踪 token 来源，生命周期关闭使用每次启动的命令行实例 id 作为所有权证明。

**技术栈：** Node.js 内置模块（`http`、`fs`、`path`、`crypto`）、现有 `ws` 测试依赖、Bash 脚本、Windows 上的 Git Bash、用于 PR 元数据的 `gh` CLI。

**提交纪律：** 每个任务都包含建议的提交。使用子代理驱动执行时，编排者审阅工作者 diff、运行任务验证并执行提交。

---

## 文件清单

- 修改：`skills/brainstorming/scripts/server.cjs`
  - 通过 `isRegularFileInsideContentDir()` 过滤根屏幕候选项。
  - 跟踪 token 来源，并在回退时轮换或失败关闭。
- 修改：`skills/brainstorming/scripts/start-server.sh`
  - 生成 `state/server-instance-id`。
  - 在 `server.cjs` 之后传入 `--brainstorm-server-id=<id>`。
- 修改：`skills/brainstorming/scripts/stop-server.sh`
  - 在向 PID 发信号之前要求精确的 instance-id argv 证明。
  - 在 stale/stopped 结果上移除陈旧的 `server.pid` 和 `server-instance-id`。
- 修改：`tests/brainstorm-server/server.test.js`
  - 添加固定端口启动防护。
  - 添加支持跳过的符号链接能力测试 harness。
  - 添加根符号链接与硬链接逃逸回归测试。
- 修改：`tests/brainstorm-server/auth.test.js`
  - 添加固定端口启动防护。
- 修改：`tests/brainstorm-server/lifecycle.test.js`
  - 添加回退 token 轮换、显式 token 失败关闭、回退键拒绝的回归测试。
- 修改：`tests/brainstorm-server/stop-server.test.sh`
  - 添加顶层清理 trap。
  - 添加正向和负向的 server-instance-id 所有权测试。
- 修改：`tests/brainstorm-server/start-server.test.sh`
  - 断言类 Windows 的 fake-node 路径接收到确切的 server id argv，并写入有效的 id 文件。
- 修改：`tests/brainstorm-server/windows-lifecycle.test.sh`
  - 为直接 Node stop-server 覆盖传入 server id argv。
  - 添加针对该 id argv 的 Windows fake-node 断言。
- 修改：`skills/brainstorming/visual-companion.md`
  - 在应保留自动打开行为的平台命令中加入 `--open`。
- 修改：`docs/superpowers/plans/2026-06-09-visual-companion-issues.md`
  - 核对已交付范围、WS Origin 措辞、默认超时和延期功能项。
- 在受跟踪文件之外更新：PR #1720 正文
  - 记录 rebase 后的 diff 状态、RED/GREEN 证据、macOS/Windows 验证、手动浏览器冒烟和外部 eval 证据。

## Task 0：Rebase 与基线状态

**文件：**
- 无源码改动
- 验证目标：git 分支状态

- [ ] **Step 1：拉取当前 dev**

运行：

```bash
git fetch origin dev
```

预期：命令以 0 退出。

- [ ] **Step 2：Rebase 到当前 dev**

运行：

```bash
git rebase origin/dev
```

预期：命令以 0 退出，或仅在需要解决的冲突处停下，对于 `evals` 必须采用 `origin/dev` 的版本。

- [ ] **Step 3：通过采用 dev 解决 evals 冲突**

如果 rebase 停在 `evals`，运行：

```bash
git restore --source=origin/dev --staged --worktree evals
git add evals
git rebase --continue
```

预期：rebase 继续。rebase 完成后，`git diff --name-only origin/dev...HEAD -- evals` 不打印任何内容。

- [ ] **Step 4：记录基线状态**

运行：

```bash
git status --short --branch
git diff --name-only origin/dev...HEAD -- evals
```

预期：状态显示分支位于 `origin/dev` 之上；第二条命令不打印任何路径。

## Task 1：根屏幕包含

**文件：**
- 修改：`tests/brainstorm-server/server.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **Step 1：添加固定端口防护和支持跳过的测试 helper**

在 `tests/brainstorm-server/server.test.js` 中，在 `waitForServer()` 之后添加此 helper：

```js
class SkipTest extends Error {
  constructor(message) {
    super(message);
    this.skip = true;
  }
}

function skip(message) {
  throw new SkipTest(message);
}

function serverStartedMessage(out) {
  const line = out.trim().split('\n').find(l => l.includes('server-started'));
  assert(line, 'server-started JSON should be present');
  return JSON.parse(line);
}

function assertStartedOnExpectedPort(out) {
  const msg = serverStartedMessage(out);
  assert.strictEqual(
    msg.port,
    TEST_PORT,
    `server.test.js expected fixed port ${TEST_PORT}, got ${msg.port}; fixed-port tests must not run through fallback`
  );
  return msg;
}

function ensureSymlinkWorks(target, link) {
  try {
    fs.symlinkSync(target, link);
    fs.unlinkSync(link);
  } catch (e) {
    try { fs.unlinkSync(link); } catch (ignore) {}
    skip(`symlink creation unavailable on this host: ${e.message}`);
  }
}
```

然后将启动段从：

```js
  const { stdout: initialStdout } = await waitForServer(server);
  let passed = 0;
  let failed = 0;
```

改为：

```js
  const { stdout: initialStdout } = await waitForServer(server);
  assertStartedOnExpectedPort(initialStdout);
  let passed = 0;
  let failed = 0;
  let skipped = 0;
```

修改 `test()` helper 的 catch 块以处理跳过：

```js
    }).catch(e => {
      if (e && e.skip) {
        console.log(`  SKIP: ${name}`);
        console.log(`    ${e.message}`);
        skipped++;
        return;
      }
      console.log(`  FAIL: ${name}`);
      console.log(`    ${e.message}`);
      failed++;
    });
```

将汇总行改为：

```js
    console.log(`\n--- Results: ${passed} passed, ${failed} failed, ${skipped} skipped ---`);
```

- [ ] **Step 2：让现有 `/files/*` 符号链接测试支持跳过**

将 `does not serve symlinks that escape content dir via /files/` 内部的设置替换为：

```js
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'linked-server-info.txt');
      try { fs.unlinkSync(link); } catch (e) {}
      ensureSymlinkWorks(target, link);
      fs.symlinkSync(target, link);
```

预期行为：无法创建可用符号链接的主机仅跳过此断言。

- [ ] **Step 3：为根符号链接和硬链接逃逸添加 RED 测试**

在现有 `/files/*` 硬链接测试之后添加这些测试：

```js
    await test('does not serve symlinks that escape content dir via root screen selection', async () => {
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'root-linked-server-info.html');
      try { fs.unlinkSync(link); } catch (e) {}
      ensureSymlinkWorks(target, link);
      fs.symlinkSync(target, link);
      const future = new Date(Date.now() + 2000);
      fs.utimesSync(target, future, future);
      await sleep(300);

      const res = await fetch(`http://localhost:${TEST_PORT}/`);
      assert.strictEqual(res.status, 200);
      assert(!res.body.includes('"type":"server-started"'), 'root screen must not serve state/server-info through a symlink');
      assert(!res.body.includes('"state_dir"'), 'root screen must not include server-info body');
    });

    await test('does not serve hard links that escape content dir via root screen selection', async () => {
      const target = path.join(STATE_DIR, 'server-info');
      const link = path.join(CONTENT_DIR, 'root-hard-linked-server-info.html');
      try { fs.unlinkSync(link); } catch (e) {}
      try {
        fs.linkSync(target, link);
      } catch (e) {
        skip(`hardlink creation unavailable on this host: ${e.message}`);
      }
      const linkStat = fs.lstatSync(link);
      if (linkStat.nlink <= 1) {
        skip(`hardlink nlink did not expose multiple links: ${linkStat.nlink}`);
      }
      const future = new Date(Date.now() + 3000);
      fs.utimesSync(target, future, future);
      await sleep(300);

      const res = await fetch(`http://localhost:${TEST_PORT}/`);
      assert.strictEqual(res.status, 200);
      assert(!res.body.includes('"type":"server-started"'), 'root screen must not serve state/server-info through a hardlink');
      assert(!res.body.includes('"state_dir"'), 'root screen must not include server-info body');
    });
```

- [ ] **Step 4：验证 RED**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node server.test.js
```

预期：在生产修复之前，至少有一个新的根包含测试失败，因为根屏幕选择可以读取 `state/server-info`。

- [ ] **Step 5：实现根包含**

在 `skills/brainstorming/scripts/server.cjs` 中，将 `getNewestScreen()` 替换为：

```js
function getNewestScreen() {
  const files = fs.readdirSync(CONTENT_DIR)
    .filter(f => !f.startsWith('.') && f.endsWith('.html'))
    .map(f => {
      const fp = path.join(CONTENT_DIR, f);
      if (!isRegularFileInsideContentDir(fp)) return null;
      return { path: fp, mtime: fs.statSync(fp).mtime.getTime() };
    })
    .filter(Boolean)
    .sort((a, b) => b.mtime - a.mtime);
  return files.length > 0 ? files[0].path : null;
}
```

- [ ] **Step 6：验证 GREEN**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node server.test.js
```

预期：根符号链接和受支持的硬链接测试通过，或仅在不支持的主机能力下跳过。现有 `/files/*` 包含测试保持通过。

- [ ] **Step 7：提交**

运行：

```bash
git add tests/brainstorm-server/server.test.js skills/brainstorming/scripts/server.cjs
git commit -m "Harden root screen containment"
```

## Task 2：回退 Token 隔离

**文件：**
- 修改：`tests/brainstorm-server/lifecycle.test.js`
- 修改：`skills/brainstorming/scripts/server.cjs`

- [ ] **Step 1：添加 HTTP 状态 helper**

在 `tests/brainstorm-server/lifecycle.test.js` 中，在 `openCaptureCommand()` 之后添加此 helper：

```js
function httpStatus(port, key) {
  return new Promise(resolve => {
    const pathWithKey = key ? '/?key=' + encodeURIComponent(key) : '/';
    require('http')
      .get({ hostname: '127.0.0.1', port, path: pathWithKey }, res => {
        res.resume();
        resolve(res.statusCode);
      })
      .on('error', () => resolve(0));
  });
}
```

- [ ] **Step 2：为持久化 token 回退轮换添加 RED 测试**

在 `falls back to a random port when the preferred port is taken` 之后添加此测试：

```js
  await test('fallback with persisted token generates a fresh unpersisted key', async () => {
    const dir = fs.mkdtempSync('/tmp/bs-port-');
    const portFile = path.join(dir, '.last-port');
    const tokenFile = path.join(dir, '.last-token');
    const preferredToken = 'abababababababababababababababab';
    let a = null, b = null;

    try {
      a = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'a'),
          BRAINSTORM_PORT: 3422,
          BRAINSTORM_TOKEN: preferredToken,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outA = ''; a.stdout.on('data', d => outA += d.toString());
      for (let i = 0; i < 60 && !outA.includes('server-started'); i++) await sleep(50);
      assert(outA.includes('server-started'), 'preferred-port server should start');

      fs.writeFileSync(portFile, '3422');
      fs.writeFileSync(tokenFile, preferredToken, { mode: 0o600 });

      b = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'b'),
          BRAINSTORM_PORT_FILE: portFile,
          BRAINSTORM_TOKEN_FILE: tokenFile,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outB = ''; b.stdout.on('data', d => outB += d.toString());
      for (let i = 0; i < 60 && !outB.includes('server-started'); i++) await sleep(50);
      const infoB = firstServerStarted(outB);
      const fallbackKey = new URL(infoB.url).searchParams.get('key');
      const persistedAfter = fs.readFileSync(tokenFile, 'utf8').trim();
      const originalStatus = await httpStatus(3422, fallbackKey);

      assert.notStrictEqual(infoB.port, 3422, 'fallback should use a different port');
      assert.notStrictEqual(fallbackKey, preferredToken, 'fallback must not reuse persisted key');
      assert.strictEqual(persistedAfter, preferredToken, 'fallback must not overwrite .last-token');
      assert.strictEqual(originalStatus, 403, 'fallback key must not authenticate to original server');
    } finally {
      await killAndWait(a);
      await killAndWait(b);
      fs.rmSync(dir, { recursive: true, force: true });
    }
  });
```

- [ ] **Step 3：为显式 token 回退失败关闭添加 RED 测试**

在持久化 token 回退测试之后立即添加此测试：

```js
  await test('fallback with explicit BRAINSTORM_TOKEN fails closed', async () => {
    const dir = fs.mkdtempSync('/tmp/bs-port-');
    const portFile = path.join(dir, '.last-port');
    const explicitToken = 'cdcdcdcdcdcdcdcdcdcdcdcdcdcdcdcd';
    let a = null, b = null;

    try {
      a = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'a'),
          BRAINSTORM_PORT: 3423,
          BRAINSTORM_TOKEN: explicitToken,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outA = ''; a.stdout.on('data', d => outA += d.toString());
      for (let i = 0; i < 60 && !outA.includes('server-started'); i++) await sleep(50);
      assert(outA.includes('server-started'), 'preferred-port server should start');

      fs.writeFileSync(portFile, '3423');
      b = spawn('node', [SERVER], {
        env: {
          ...process.env,
          BRAINSTORM_DIR: path.join(dir, 'b'),
          BRAINSTORM_PORT_FILE: portFile,
          BRAINSTORM_TOKEN: explicitToken,
          BRAINSTORM_LIFECYCLE_CHECK_MS: 100000
        }
      });
      let outB = ''; let errB = '';
      b.stdout.on('data', d => outB += d.toString());
      b.stderr.on('data', d => errB += d.toString());
      for (let i = 0; i < 60 && !outB.includes('server-started') && b.exitCode === null; i++) await sleep(50);
      const exited = await waitForExit(b, 1500);

      assert(exited, 'explicit-token fallback process should exit');
      assert.notStrictEqual(b.exitCode, 0, 'explicit-token fallback should fail non-zero');
      assert(!outB.includes('server-started'), 'explicit-token fallback must not start on a random port');
      assert(/BRAINSTORM_TOKEN/.test(errB), `stderr should explain explicit token fallback refusal, got: ${errB}`);
    } finally {
      await killAndWait(a);
      await killAndWait(b);
      fs.rmSync(dir, { recursive: true, force: true });
    }
  });
```

- [ ] **Step 4：验证 RED**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node lifecycle.test.js
```

预期：持久化 token 回退测试失败，因为回退复用 `.last-token`；显式 token 回退测试失败，因为回退当前会启动。

- [ ] **Step 5：在生产代码中跟踪 token 来源**

在 `skills/brainstorming/scripts/server.cjs` 中，将当前的 `const TOKEN = (() => { ... })();` 块替换为：

```js
function generateToken() {
  return crypto.randomBytes(32).toString('hex');
}

function initialToken() {
  if (process.env.BRAINSTORM_TOKEN) {
    return { value: process.env.BRAINSTORM_TOKEN, source: 'env' };
  }
  if (TOKEN_FILE) {
    try {
      const t = fs.readFileSync(TOKEN_FILE, 'utf-8').trim();
      if (/^[0-9a-f]{32,}$/i.test(t)) return { value: t, source: 'file' };
    } catch (e) { /* no prior token recorded */ }
  }
  return { value: generateToken(), source: 'generated' };
}

const tokenInfo = initialToken();
let TOKEN = tokenInfo.value;
let tokenSource = tokenInfo.source;
```

- [ ] **Step 6：在 EADDRINUSE 回退时轮换或失败关闭**

在 `server.on('error', ...)` 处理器中，将 `EADDRINUSE` 分支替换为：

```js
    if (err.code === 'EADDRINUSE' && !triedFallback) {
      if (tokenSource === 'env') {
        console.error('Server failed to bind: preferred port is in use and BRAINSTORM_TOKEN is set; refusing fallback with explicit token');
        process.exit(1);
      }
      triedFallback = true;
      PORT = randomPort();
      if (tokenSource === 'file') {
        TOKEN = generateToken();
        tokenSource = 'generated-fallback';
      }
      server.listen(PORT, HOST, onListen);
    } else {
```

- [ ] **Step 7：验证 GREEN**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node lifecycle.test.js
```

预期：所有生命周期测试通过，包括回退 token 轮换和显式 token 失败关闭。

- [ ] **Step 8：提交**

运行：

```bash
git add tests/brainstorm-server/lifecycle.test.js skills/brainstorming/scripts/server.cjs
git commit -m "Isolate companion fallback tokens"
```

## Task 3：Stop-Server 的 Instance-Id 所有权

**文件：**
- 修改：`tests/brainstorm-server/stop-server.test.sh`
- 修改：`skills/brainstorming/scripts/start-server.sh`
- 修改：`skills/brainstorming/scripts/stop-server.sh`

- [ ] **Step 1：向 stop-server 测试添加清理跟踪和 id helper**

在 `tests/brainstorm-server/stop-server.test.sh` 中，在 `PASS=0; FAIL=0` 之后添加：

```bash
PIDS=()
DIRS=()

cleanup() {
  for pid in "${PIDS[@]}"; do
    kill -9 "$pid" 2>/dev/null || true
    wait "$pid" 2>/dev/null || true
  done
  for dir in "${DIRS[@]}"; do
    rm -rf "$dir"
  done
}
trap cleanup EXIT

track_dir() { DIRS+=("$1"); }
track_pid() { PIDS+=("$1"); }
new_server_id() {
  printf 'testid%026d\n' "$RANDOM"
}
```

当每个测试创建 `SESS="$(mktemp -d)"` 时，立即添加：

```bash
track_dir "$SESS"
```

当测试启动 `UNRELATED`、`SRV` 或 `IMPOSTOR` 时，立即添加对应的跟踪调用：

```bash
track_pid "$UNRELATED"
track_pid "$SRV"
track_pid "$IMPOSTOR"
```

- [ ] **Step 2：添加 RED 所有权测试**

将当前的真实服务器和冒名者段落替换为以下用例：

```bash
# --- Test 2: a real brainstorm server with matching instance id IS stopped ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/content" "$SESS/state"
SERVER_ID="$(new_server_id)"
printf '%s\n' "$SERVER_ID" > "$SESS/state/server-instance-id"
BRAINSTORM_DIR="$SESS" BRAINSTORM_PORT=3399 node "$SERVER" "--brainstorm-server-id=$SERVER_ID" > /dev/null 2>&1 &
SRV=$!
track_pid "$SRV"
disown "$SRV" 2>/dev/null || true
for _ in $(seq 1 40); do kill -0 "$SRV" 2>/dev/null && break; sleep 0.1; done
sleep 0.4
echo "$SRV" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
sleep 0.3
if kill -0 "$SRV" 2>/dev/null; then
  bad "real brainstorm server still running after stop" "$OUT"
else
  case "$OUT" in
    *stopped*) ok "real brainstorm server with matching instance id is stopped" ;;
    *) bad "server stopped but status was not 'stopped'" "$OUT" ;;
  esac
fi

# --- Test 4: a node server.cjs impostor with missing instance id is spared ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/state"
( exec -a "node server.cjs" sleep 600 ) &
IMPOSTOR=$!
track_pid "$IMPOSTOR"
disown "$IMPOSTOR" 2>/dev/null || true
echo "$IMPOSTOR" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
if kill -0 "$IMPOSTOR" 2>/dev/null; then
  case "$OUT" in
    *stale_pid*) ok "missing instance id leaves node server.cjs impostor alone" ;;
    *) bad "impostor survived but status was not stale_pid" "$OUT" ;;
  esac
else
  bad "killed a node server.cjs impostor with missing instance id" "$OUT"
fi

# --- Test 5: a node server.cjs impostor with wrong instance id is spared ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/state"
EXPECTED_ID="$(new_server_id)"
WRONG_ID="$(new_server_id)"
printf '%s\n' "$EXPECTED_ID" > "$SESS/state/server-instance-id"
( exec -a "node server.cjs --brainstorm-server-id=$WRONG_ID" sleep 600 ) &
IMPOSTOR=$!
track_pid "$IMPOSTOR"
disown "$IMPOSTOR" 2>/dev/null || true
echo "$IMPOSTOR" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
if kill -0 "$IMPOSTOR" 2>/dev/null; then
  case "$OUT" in
    *stale_pid*) ok "wrong instance id leaves node server.cjs impostor alone" ;;
    *) bad "wrong-id impostor survived but status was not stale_pid" "$OUT" ;;
  esac
else
  bad "killed a node server.cjs impostor with wrong instance id" "$OUT"
fi

# --- Test 6: malformed instance id is fail-closed ---
SESS="$(mktemp -d)"; track_dir "$SESS"; mkdir -p "$SESS/state"
printf '%s\n' 'bad id with spaces' > "$SESS/state/server-instance-id"
( exec -a "node server.cjs --brainstorm-server-id=bad-id-with-spaces" sleep 600 ) &
IMPOSTOR=$!
track_pid "$IMPOSTOR"
disown "$IMPOSTOR" 2>/dev/null || true
echo "$IMPOSTOR" > "$SESS/state/server.pid"
OUT="$("$STOP" "$SESS")"
if kill -0 "$IMPOSTOR" 2>/dev/null; then
  case "$OUT" in
    *stale_pid*) ok "malformed instance id is fail-closed" ;;
    *) bad "malformed-id impostor survived but status was not stale_pid" "$OUT" ;;
  esac
else
  bad "killed process despite malformed instance id" "$OUT"
fi
```

保留无关 PID 和缺失 PID 的测试。

- [ ] **Step 3：验证 RED**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers
bash tests/brainstorm-server/stop-server.test.sh
```

预期：在实现之前，匹配 instance id 的真实服务器被报告为 `stale_pid`，且其中一个冒名者用例可能被旧的命令名证明杀死。

- [ ] **Step 4：在 start-server 中生成并传入 instance id**

在 `skills/brainstorming/scripts/start-server.sh` 中，在 `LOG_FILE="${STATE_DIR}/server.log"` 之后添加：

```bash
SERVER_ID_FILE="${STATE_DIR}/server-instance-id"
```

在 `mkdir -p "${SESSION_DIR}/content" "$STATE_DIR"` 之后添加：

```bash
SERVER_ID=""
if [[ -r /dev/urandom ]]; then
  SERVER_ID="$(od -An -N24 -tx1 /dev/urandom 2>/dev/null | tr -d ' \n' || true)"
fi
if ! [[ "$SERVER_ID" =~ ^[A-Za-z0-9_-]{32,64}$ ]]; then
  SERVER_ID="$(printf '%08x%08x%08x%08x' "$$" "$(date +%s)" "${RANDOM:-0}" "${RANDOM:-0}")"
fi
printf '%s\n' "$SERVER_ID" > "$SERVER_ID_FILE"
chmod 600 "$SERVER_ID_FILE" 2>/dev/null || true
```

将两条 Node 启动命令更新为传入 argv：

```bash
env BRAINSTORM_DIR="$SESSION_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" BRAINSTORM_OWNER_PID="$OWNER_PID" node server.cjs "--brainstorm-server-id=$SERVER_ID" &
```

以及：

```bash
nohup env BRAINSTORM_DIR="$SESSION_DIR" BRAINSTORM_HOST="$BIND_HOST" BRAINSTORM_URL_HOST="$URL_HOST" BRAINSTORM_OWNER_PID="$OWNER_PID" node server.cjs "--brainstorm-server-id=$SERVER_ID" > "$LOG_FILE" 2>&1 &
```

- [ ] **Step 5：在 stop-server 中要求 instance id**

在 `skills/brainstorming/scripts/stop-server.sh` 中，添加：

```bash
SERVER_ID_FILE="${STATE_DIR}/server-instance-id"
```

将 `is_brainstorm_server()` 替换为：

```bash
read_expected_server_id() {
  [[ -f "$SERVER_ID_FILE" ]] || return 1
  local id
  id="$(tr -d '\r\n' < "$SERVER_ID_FILE" 2>/dev/null || true)"
  [[ "$id" =~ ^[A-Za-z0-9_-]{32,64}$ ]] || return 1
  printf '%s\n' "$id"
}

command_line_for_pid() {
  local pid="$1"
  if [[ -r "/proc/$pid/cmdline" ]]; then
    tr '\0' '\n' < "/proc/$pid/cmdline" 2>/dev/null || true
    return 0
  fi
  ps -ww -p "$pid" -o command= 2>/dev/null || ps -f -p "$pid" 2>/dev/null | sed '1d' || true
}

command_has_server_id() {
  local pid="$1"
  local expected="$2"
  local expected_arg="--brainstorm-server-id=$expected"
  if [[ -r "/proc/$pid/cmdline" ]]; then
    local arg
    while IFS= read -r -d '' arg; do
      [[ "$arg" == "$expected_arg" ]] && return 0
    done < "/proc/$pid/cmdline"
    return 1
  fi
  local command_line
  command_line="$(command_line_for_pid "$pid")"
  [[ -n "$command_line" ]] || return 1
  case " $command_line " in
    *" $expected_arg "*) return 0 ;;
    *) return 1 ;;
  esac
}

is_brainstorm_server() {
  kill -0 "$1" 2>/dev/null || return 1
  local expected_id
  expected_id="$(read_expected_server_id)" || return 1
  command_has_server_id "$1" "$expected_id" || return 1
  return 0
}
```

在 stale PID 分支中，移除两个元数据文件：

```bash
    rm -f "$PID_FILE" "$SERVER_ID_FILE"
```

在 stopped 分支中，将清理行改为：

```bash
  rm -f "$PID_FILE" "$SERVER_ID_FILE" "${STATE_DIR}/server.log"
```

- [ ] **Step 6：验证 GREEN**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers
bash tests/brainstorm-server/stop-server.test.sh
```

预期：真实匹配 id 的服务器停止，冒名者存活，所有 stale 用例返回 `stale_pid`。

- [ ] **Step 7：提交**

运行：

```bash
git add tests/brainstorm-server/stop-server.test.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh
git commit -m "Harden companion stop ownership proof"
```

## Task 4：平台与固定端口测试加固

**文件：**
- 修改：`tests/brainstorm-server/auth.test.js`
- 修改：`tests/brainstorm-server/start-server.test.sh`
- 修改：`tests/brainstorm-server/windows-lifecycle.test.sh`

- [ ] **Step 1：向 auth 测试添加固定端口防护**

在 `tests/brainstorm-server/auth.test.js` 中，在 `waitForServer()` 之后添加此 helper：

```js
function serverStartedMessage(out) {
  const line = out.trim().split('\n').find(l => l.includes('server-started'));
  assert(line, 'server-started JSON should be present');
  return JSON.parse(line);
}

function assertStartedOnExpectedPort(out) {
  const msg = serverStartedMessage(out);
  assert.strictEqual(
    msg.port,
    TEST_PORT,
    `auth.test.js expected fixed port ${TEST_PORT}, got ${msg.port}; fixed-port tests must not run through fallback`
  );
  return msg;
}
```

在 `const { stdout: initialStdout } = await waitForServer(server);` 之后添加：

```js
  assertStartedOnExpectedPort(initialStdout);
```

- [ ] **Step 2：验证 auth 固定端口防护**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node auth.test.js
```

预期：auth 测试在空闲的 `3335` 上通过，并在发生回退时明确失败。

- [ ] **Step 3：添加 start-server id argv 断言**

在 `tests/brainstorm-server/start-server.test.sh` 中，将第一个 fake node 主体改为：

```bash
cat > "$TEST_DIR/fake-bin/node" <<'EOF'
#!/usr/bin/env bash
echo "CAPTURED_OWNER_PID=${BRAINSTORM_OWNER_PID:-__UNSET__}"
echo "CAPTURED_ARGV=$*"
exit 0
EOF
```

在 owner PID 断言之后添加：

```bash
captured_argv=$(echo "$captured" | grep "CAPTURED_ARGV=" | head -1 | sed 's/CAPTURED_ARGV=//')
if echo "$captured_argv" | grep -Eq -- '--brainstorm-server-id=[A-Za-z0-9_-]{32,64}'; then
  pass "passes shell-safe server instance id argv"
else
  fail "passes shell-safe server instance id argv" \
       "expected --brainstorm-server-id=<safe id>, got: $captured_argv"
fi

server_id_file=$(find "$TEST_DIR/project/.superpowers/brainstorm" -name server-instance-id -print 2>/dev/null | head -1)
server_id_value=""
if [[ -n "$server_id_file" ]]; then
  server_id_value="$(tr -d '\r\n' < "$server_id_file")"
fi
if [[ "$server_id_value" =~ ^[A-Za-z0-9_-]{32,64}$ ]]; then
  pass "writes shell-safe server-instance-id state file"
else
  fail "writes shell-safe server-instance-id state file" \
       "expected valid id in state, got '$server_id_value'"
fi
```

- [ ] **Step 4：添加 Windows 生命周期 id argv 断言**

在 `tests/brainstorm-server/windows-lifecycle.test.sh` 中，将 Test 2 的 fake node 主体改为：

```bash
cat > "$FAKE_NODE_DIR/node" <<'FAKENODE'
#!/usr/bin/env bash
echo "CAPTURED_OWNER_PID=${BRAINSTORM_OWNER_PID:-__UNSET__}"
echo "CAPTURED_ARGV=$*"
exit 0
FAKENODE
```

在 Test 2 的 owner PID 检查之后添加：

```bash
captured_argv=$(echo "$captured" | grep "CAPTURED_ARGV=" | head -1 | sed 's/CAPTURED_ARGV=//')
if echo "$captured_argv" | grep -Eq -- '--brainstorm-server-id=[A-Za-z0-9_-]{32,64}'; then
  pass "start-server.sh passes server instance id argv on Windows"
else
  fail "start-server.sh passes server instance id argv on Windows" \
       "Expected --brainstorm-server-id=<safe id>, output: $captured"
fi
```

在 Test 6 中，启动直接 Node 之前，添加：

```bash
STOP_TEST_ID="$(printf 'windowsstop%021d\n' "$RANDOM")"
printf '%s\n' "$STOP_TEST_ID" > "$TEST_DIR/stop-test/state/server-instance-id"
```

将 Test 6 中的直接 Node 启动改为：

```bash
  node "$SERVER_SCRIPT" "--brainstorm-server-id=$STOP_TEST_ID" > "$TEST_DIR/stop-test/.server.log" 2>&1 &
```

- [ ] **Step 5：验证平台测试**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers
bash tests/brainstorm-server/start-server.test.sh
```

预期：所有 start-server shell 测试在 macOS 上通过。

Windows 生命周期测试稍后在 `ballmer` 上作为 Task 6 的一部分运行。

- [ ] **Step 6：提交**

运行：

```bash
git add tests/brainstorm-server/auth.test.js tests/brainstorm-server/start-server.test.sh tests/brainstorm-server/windows-lifecycle.test.sh
git commit -m "Harden companion platform tests"
```

## Task 5：文档与 PR 一致性

**文件：**
- 修改：`skills/brainstorming/visual-companion.md`
- 修改：`docs/superpowers/plans/2026-06-09-visual-companion-issues.md`
- 更新：通过 `gh pr edit` 更新 PR #1720 正文

- [ ] **Step 1：保持平台启动命令与自动打开行为一致**

在 `skills/brainstorming/visual-companion.md` 中，更新启动用户批准的伴侣会话的平台特定命令，使其包含 `--open`：

```bash
scripts/start-server.sh --project-dir /path/to/project --open
```

```bash
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

不要在有意跳过自动打开的远程绑定示例中添加 `--open`。

- [ ] **Step 2：核对问题目录处置行**

在 `docs/superpowers/plans/2026-06-09-visual-companion-issues.md` 中，将 A2、D1、D2、D3 和 D4 的处置行替换为：

```markdown
| A2 | Host allowlist; browser WS Origin check | PRs #1110/#1553 | Host allowlist dropped; WS Origin check retained after auth for browser confused-deputy defense |
| D1 | Permanent opt-out of the companion | issue #892 | Deferred - not in PR #1720 |
| D2 | Free-text feedback from the browser | issue #957 | Deferred - not in PR #1720 |
| D3 | Auto-open the companion URL | PR #759 (#755) | Done in PR #1720 via `--open` |
| D4 | Light/dark contrast helpers in the frame | PR #1683 | Deferred - not in PR #1720 |
```

- [ ] **Step 3：核对 A2 细节文本**

将 A2 段落的最后一句话替换为：

```markdown
No `BRAINSTORM_ALLOWED_HOSTS` and no Host allowlist. The final implementation still checks browser WebSocket `Origin` after session auth so a cross-origin localhost tab cannot ride the companion cookie.
```

- [ ] **Step 4：核对超时和功能分组文本**

在 C1 段落中，将：

```markdown
- Raise the default (about 2h) and make it configurable:
```

替换为：

```markdown
- Raise the default to 4 hours and make it configurable:
```

在建议分组段落中，将第 4 项替换为：

```markdown
4. **Deferred feature pass** - D1, D2, D4 are not part of PR #1720. D3 is shipped through the `--open` flow.
```

- [ ] **Step 5：验证文档 diff**

运行：

```bash
git diff -- skills/brainstorming/visual-companion.md docs/superpowers/plans/2026-06-09-visual-companion-issues.md
```

预期：diff 仅更新自动打开命令一致性、已交付/延期处置、WS Origin 措辞，以及 4 小时超时陈述。

- [ ] **Step 6：提交**

运行：

```bash
git add skills/brainstorming/visual-companion.md docs/superpowers/plans/2026-06-09-visual-companion-issues.md
git commit -m "Align visual companion docs with shipped scope"
```

## Task 6：完整验证与证据

**文件：**
- 无必需源码改动
- 更新：PR #1720 正文

- [ ] **Step 1：运行聚焦的 macOS 检查**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
node server.test.js
node auth.test.js
node lifecycle.test.js
bash stop-server.test.sh
bash start-server.test.sh
```

预期：所有聚焦测试通过；仅在主机不支持时，符号链接专属测试可能报告跳过。

- [ ] **Step 2：运行完整 macOS 测试套件**

运行：

```bash
cd /Users/drewritter/.codex/worktrees/59f6/superpowers/tests/brainstorm-server
npm test
```

预期：完整 brainstorm-server 测试套件通过。

- [ ] **Step 3：运行静态检查**

从仓库根目录运行：

```bash
git diff --check
node --check skills/brainstorming/scripts/server.cjs
node --check skills/brainstorming/scripts/helper.js
bash scripts/lint-shell.sh skills/brainstorming/scripts/start-server.sh skills/brainstorming/scripts/stop-server.sh tests/brainstorm-server/start-server.test.sh tests/brainstorm-server/stop-server.test.sh tests/brainstorm-server/windows-lifecycle.test.sh
```

预期：所有命令以 0 退出。

- [ ] **Step 4：在 ballmer 上运行 Windows 验证**

在 `ballmer` 上复制或拉取已 rebase 的分支，然后运行：

```bash
cd superpowers
npm --prefix tests/brainstorm-server ci
npm --prefix tests/brainstorm-server test
bash tests/brainstorm-server/windows-lifecycle.test.sh
```

预期：完整可运行的 Windows 套件通过。如果 Git Bash 缺少 `lsof`，仅 lsof 专属的遗留端口交叉检查测试可能跳过；instance-id 停止测试仍必须通过。

- [ ] **Step 5：验证 PR diff 与 GitHub 状态**

运行：

```bash
git diff --quiet origin/dev...HEAD -- evals
gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid
```

预期：第一条命令以 0 退出。分支推送后 PR JSON 不再报告 `DIRTY` 或 `CONFLICTING`。

- [ ] **Step 6：收集外部 eval 证据**

运行：

```bash
git -C /Users/drewritter/.codex/worktrees/59f6/superpowers-evals rev-parse HEAD
git -C /Users/drewritter/.codex/worktrees/59f6/superpowers-evals status --short --branch
```

如果 eval worktree 不在该路径，在 `/Users/drewritter/prime-rad/superpowers-evals` 中运行相同命令。

从已运行的 eval 证据中记录确切的 eval 场景路径、命令、结果产物路径和 RED/GREEN 结果。不要声称 eval 子模块包含在 PR #1720 中。

- [ ] **Step 7：运行最终手动/浏览器冒烟**

在自动化测试通过后，使用 `--open` 启动伴侣、推送一个小屏幕、验证浏览器在 bootstrap 后到达裸 `/` URL、验证状态到达 Connected、使用相同项目目录停止并重启服务器、验证打开的标签页重新连接。记录确切的命令和观察到的结果。

- [ ] **Step 8：更新 PR 正文**

准备 `/tmp/pr-1720-body.md`，然后在该正文包含以下内容后运行 `gh pr edit 1720 --body-file /tmp/pr-1720-body.md`：

- 模型、宿主、插件，以及 Drew 作为人类审阅者
- 重复/相关 PR 搜索结果
- rebase 后明确说明 `evals` 不在本 PR diff 中
- 聚焦的 RED/GREEN 证据表
- macOS `npm test` 证据
- Windows `ballmer` 证据
- 手动/浏览器冒烟证据
- 外部 eval 仓库提交、场景路径、命令、产物路径和结果

- [ ] **Step 9：推送分支**

运行：

```bash
git status --short --branch
git push origin brainstorming-companion
```

预期：推送成功且 PR #1720 更新。

- [ ] **Step 10：最终 PR 就绪检查**

运行：

```bash
gh pr view 1720 --json mergeStateStatus,statusCheckRollup,headRefOid,url
```

预期：PR 指向已推送的 head SHA，合并状态不再被冲突阻塞，检查状态已为 Drew 记录。

## 自审清单

- [ ] `docs/superpowers/specs/2026-06-11-visual-companion-final-hardening-fixup-design.md` 中的每一条要求都映射到上述任务之一。
- [ ] 本计划不包含含糊或不完整的步骤。
- [ ] Task 1、2、3 中测试先于生产修复添加。
- [ ] 文档任务不添加延期功能。
- [ ] 验证任务包含 macOS、Windows、PR diff、PR 元数据、外部 eval 证据和最终手动/浏览器冒烟。
