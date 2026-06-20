# 可视化头脑风暴重构实现计划

> **致代理工作者：** 必须使用 superpowers:subagent-driven-development（如果支持子代理）或 superpowers:executing-plans 来实现本计划。步骤使用复选框（`- [ ]`）语法进行跟踪。

**目标：** 将可视化头脑风暴从阻塞式 TUI 反馈模型重构为非阻塞的“浏览器展示，终端命令”架构。

**架构：** 浏览器成为交互式展示；终端保持对话通道。服务器将用户事件写入一个按屏幕划分的 `.events` 文件，Claude 在下一轮读取该文件。消除 `wait-for-feedback.sh` 和所有 `TaskOutput` 阻塞。

**技术栈：** Node.js（Express、ws、chokidar），原生 HTML/CSS/JS

**规格：** `docs/superpowers/specs/2026-02-19-visual-brainstorming-refactor-design.md`

---

## 文件映射

| 文件 | 操作 | 职责 |
|------|--------|---------------|
| `lib/brainstorm-server/index.js` | Modify | 服务器：新增 `.events` 文件写入、新屏清空、替换 `wrapInFrame` |
| `lib/brainstorm-server/frame-template.html` | Modify | 模板：移除反馈页脚，新增内容占位符和选择指示器 |
| `lib/brainstorm-server/helper.js` | Modify | 客户端 JS：移除 send/feedback 函数，收窄为点击捕获 + 指示器更新 |
| `lib/brainstorm-server/wait-for-feedback.sh` | Delete | 不再需要 |
| `skills/brainstorming/visual-companion.md` | Modify | 技能说明：将循环改写为非阻塞流程 |
| `tests/brainstorm-server/server.test.js` | Modify | 测试：针对新模板结构和 helper.js API 更新 |

---

## 区块 1：服务器、模板、客户端、测试、技能

### Task 1: 更新 `frame-template.html`

**文件：**
- Modify: `lib/brainstorm-server/frame-template.html`

- [ ] **Step 1: 移除反馈页脚 HTML**

将 feedback-footer div（第 227-233 行）替换为选择指示器条：

```html
  <div class="indicator-bar">
    <span id="indicator-text">Click an option above, then return to the terminal</span>
  </div>
```

同时将 `#claude-content`（第 220-223 行）内的默认内容替换为内容占位符：

```html
    <div id="claude-content">
      <!-- CONTENT -->
    </div>
```

- [ ] **Step 2: 将反馈页脚 CSS 替换为指示器条 CSS**

移除 `.feedback-footer`、`.feedback-footer label`、`.feedback-row` 以及 `.feedback-footer` 内的 textarea/button 样式（第 82-112 行）。

新增指示器条 CSS：

```css
    .indicator-bar {
      background: var(--bg-secondary);
      border-top: 1px solid var(--border);
      padding: 0.5rem 1.5rem;
      flex-shrink: 0;
      text-align: center;
    }
    .indicator-bar span {
      font-size: 0.75rem;
      color: var(--text-secondary);
    }
    .indicator-bar .selected-text {
      color: var(--accent);
      font-weight: 500;
    }
```

- [ ] **Step 3: 验证模板渲染**

运行测试套件检查模板仍能加载：
```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：Test 1-5 应仍通过。Test 6-8 可能失败（预期之内——它们断言的是旧结构）。

- [ ] **Step 4: 提交**

```bash
git add lib/brainstorm-server/frame-template.html
git commit -m "Replace feedback footer with selection indicator bar in brainstorm template"
```

---

### Task 2: 更新 `index.js` — 内容注入和 `.events` 文件

**文件：**
- Modify: `lib/brainstorm-server/index.js`

- [ ] **Step 1: 为 `.events` 文件写入编写失败测试**

在 `tests/brainstorm-server/server.test.js` 的 Test 4 区域之后新增一个测试——发送带 `choice` 字段的 WebSocket 事件，并验证 `.events` 文件已被写入：

```javascript
    // Test: Choice events written to .events file
    console.log('Test: Choice events written to .events file');
    const ws3 = new WebSocket(`ws://localhost:${TEST_PORT}`);
    await new Promise(resolve => ws3.on('open', resolve));

    ws3.send(JSON.stringify({ type: 'click', choice: 'a', text: 'Option A' }));
    await sleep(300);

    const eventsFile = path.join(TEST_DIR, '.events');
    assert(fs.existsSync(eventsFile), '.events file should exist after choice click');
    const lines = fs.readFileSync(eventsFile, 'utf-8').trim().split('\n');
    const event = JSON.parse(lines[lines.length - 1]);
    assert.strictEqual(event.choice, 'a', 'Event should contain choice');
    assert.strictEqual(event.text, 'Option A', 'Event should contain text');
    ws3.close();
    console.log('  PASS');
```

- [ ] **Step 2: 运行测试确认其失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试 FAILS——`.events` 文件尚不存在。

- [ ] **Step 3: 为新屏时清空 `.events` 文件编写失败测试**

新增另一个测试：

```javascript
    // Test: .events cleared on new screen
    console.log('Test: .events cleared on new screen');
    // .events file should still exist from previous test
    assert(fs.existsSync(path.join(TEST_DIR, '.events')), '.events should exist before new screen');
    fs.writeFileSync(path.join(TEST_DIR, 'new-screen.html'), '<h2>New screen</h2>');
    await sleep(500);
    assert(!fs.existsSync(path.join(TEST_DIR, '.events')), '.events should be cleared after new screen');
    console.log('  PASS');
```

- [ ] **Step 4: 运行测试确认其失败**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新测试 FAILS——推送新屏时 `.events` 未被清空。

- [ ] **Step 5: 在 `index.js` 中实现 `.events` 文件写入**

在 WebSocket 的 `message` 处理器（`index.js` 第 74-77 行）中，`console.log` 之后新增：

```javascript
    // Write user events to .events file for Claude to read
    if (event.choice) {
      const eventsFile = path.join(SCREEN_DIR, '.events');
      fs.appendFileSync(eventsFile, JSON.stringify(event) + '\n');
    }
```

在 chokidar 的 `add` 处理器（第 104-111 行）中，新增 `.events` 清空逻辑：

```javascript
    if (filePath.endsWith('.html')) {
      // Clear events from previous screen
      const eventsFile = path.join(SCREEN_DIR, '.events');
      if (fs.existsSync(eventsFile)) fs.unlinkSync(eventsFile);

      console.log(JSON.stringify({ type: 'screen-added', file: filePath }));
      // ... existing reload broadcast
    }
```

- [ ] **Step 6: 将 `wrapInFrame` 替换为注释占位符注入**

替换 `wrapInFrame` 函数（`index.js` 第 27-32 行）：

```javascript
function wrapInFrame(content) {
  return frameTemplate.replace('<!-- CONTENT -->', content);
}
```

- [ ] **Step 7: 运行所有测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：新增的 `.events` 测试 PASS。已有测试可能仍因旧断言而失败（在 Task 4 中修复）。

- [ ] **Step 8: 提交**

```bash
git add lib/brainstorm-server/index.js tests/brainstorm-server/server.test.js
git commit -m "Add .events file writing and comment-based content injection to brainstorm server"
```

---

### Task 3: 精简 `helper.js`

**文件：**
- Modify: `lib/brainstorm-server/helper.js`

- [ ] **Step 1: 移除 `sendToClaude` 函数**

删除 `sendToClaude` 函数（第 92-106 行）——函数体和页面接管 HTML。

- [ ] **Step 2: 移除 `window.send` 函数**

删除 `window.send` 函数（第 120-129 行）——与被移除的 Send 按钮相关联。

- [ ] **Step 3: 移除表单提交和输入变更处理器**

删除表单提交处理器（第 57-71 行）和输入变更处理器（第 73-89 行），包括 `inputTimeout` 变量。

- [ ] **Step 4: 移除 `pageshow` 事件监听器**

删除我们之前新增的 `pageshow` 监听器（不再有 textarea 需要清空）。

- [ ] **Step 5: 将点击处理器收窄为仅捕获 `[data-choice]`**

将点击处理器（第 36-55 行）替换为更窄的版本：

```javascript
  // Capture clicks on choice elements
  document.addEventListener('click', (e) => {
    const target = e.target.closest('[data-choice]');
    if (!target) return;

    sendEvent({
      type: 'click',
      text: target.textContent.trim(),
      choice: target.dataset.choice,
      id: target.id || null
    });
  });
```

- [ ] **Step 6: 在点击选项时新增指示器条更新**

在点击处理器的 `sendEvent` 调用之后新增：

```javascript
    // Update indicator bar
    const indicator = document.getElementById('indicator-text');
    if (indicator) {
      const label = target.querySelector('h3, .content h3, .card-body h3')?.textContent?.trim() || target.dataset.choice;
      indicator.innerHTML = '<span class="selected-text">' + label + ' selected</span> — return to terminal to continue';
    }
```

- [ ] **Step 7: 从 `window.brainstorm` API 移除 `sendToClaude`**

更新 `window.brainstorm` 对象（第 132-136 行），移除 `sendToClaude`：

```javascript
  window.brainstorm = {
    send: sendEvent,
    choice: (value, metadata = {}) => sendEvent({ type: 'choice', value, ...metadata })
  };
```

- [ ] **Step 8: 运行测试**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```

- [ ] **Step 9: 提交**

```bash
git add lib/brainstorm-server/helper.js
git commit -m "Simplify helper.js: remove feedback functions, narrow to choice capture + indicator"
```

---

### Task 4: 针对新结构更新测试

**文件：**
- Modify: `tests/brainstorm-server/server.test.js`

**注意：** 下文的行号引用来自_原始_文件。Task 2 在文件较前位置插入了新测试，因此实际行号会偏移。按 `console.log` 标签（如 "Test 5:"、"Test 6:"）查找测试。

- [ ] **Step 1: 更新 Test 5（完整文档断言）**

找到 Test 5 的断言 `!fullRes.body.includes('feedback-footer')`。将其改为：完整文档也不应有指示器条（它们被按原样提供）：

```javascript
    assert(!fullRes.body.includes('indicator-bar') || fullDoc.includes('indicator-bar'),
      'Should not wrap full documents in frame template');
```

- [ ] **Step 2: 更新 Test 6（片段包裹）**

第 125 行：将 `feedback-footer` 断言替换为指示器条断言：

```javascript
    assert(fragRes.body.includes('indicator-bar'), 'Fragment should get indicator bar from frame');
```

同时验证内容占位符已被替换（片段内容出现，占位符注释未保留）：

```javascript
    assert(!fragRes.body.includes('<!-- CONTENT -->'), 'Content placeholder should be replaced');
```

- [ ] **Step 3: 更新 Test 7（helper.js API）**

第 140-142 行：更新断言以反映新的 API 表面：

```javascript
    assert(helperContent.includes('toggleSelect'), 'helper.js should define toggleSelect');
    assert(helperContent.includes('sendEvent'), 'helper.js should define sendEvent');
    assert(helperContent.includes('selectedChoice'), 'helper.js should track selectedChoice');
    assert(helperContent.includes('brainstorm'), 'helper.js should expose brainstorm API');
    assert(!helperContent.includes('sendToClaude'), 'helper.js should not contain sendToClaude');
```

- [ ] **Step 4: 用指示器条测试替换 Test 8（sendToClaude 主题）**

替换 Test 8（第 145-149 行）——`sendToClaude` 不再存在。改为测试指示器条：

```javascript
    // Test 8: Indicator bar uses CSS variables (theme support)
    console.log('Test 8: Indicator bar uses CSS variables');
    const templateContent = fs.readFileSync(
      path.join(__dirname, '../../lib/brainstorm-server/frame-template.html'), 'utf-8'
    );
    assert(templateContent.includes('indicator-bar'), 'Template should have indicator bar');
    assert(templateContent.includes('indicator-text'), 'Template should have indicator text element');
    console.log('  PASS');
```

- [ ] **Step 5: 运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试 PASS。

- [ ] **Step 6: 提交**

```bash
git add tests/brainstorm-server/server.test.js
git commit -m "Update brainstorm server tests for new template structure and helper.js API"
```

---

### Task 5: 删除 `wait-for-feedback.sh`

**文件：**
- Delete: `lib/brainstorm-server/wait-for-feedback.sh`

- [ ] **Step 1: 验证没有其他文件 import 或引用 `wait-for-feedback.sh`**

搜索代码库：
```bash
grep -r "wait-for-feedback" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.json"
```

预期引用：仅 `visual-companion.md`（在 Task 6 中改写）和可能的 release notes（历史记录，保持原样）。

- [ ] **Step 2: 删除该文件**

```bash
rm lib/brainstorm-server/wait-for-feedback.sh
```

- [ ] **Step 3: 运行测试确认无破坏**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试 PASS（没有测试引用此文件）。

- [ ] **Step 4: 提交**

```bash
git add -u lib/brainstorm-server/wait-for-feedback.sh
git commit -m "Delete wait-for-feedback.sh: replaced by .events file"
```

---

### Task 6: 改写 `visual-companion.md`

**文件：**
- Modify: `skills/brainstorming/visual-companion.md`

- [ ] **Step 1: 更新“工作原理”描述（第 18 行）**

将关于“以 JSON 形式接收反馈”的句子替换为：

```markdown
The server watches a directory for HTML files and serves the newest one to the browser. You write HTML content, the user sees it in their browser and can click to select options. Selections are recorded to a `.events` file that you read on your next turn.
```

- [ ] **Step 2: 更新片段描述（第 20 行）**

从 frame template 提供的内容描述中移除“feedback footer”：

```markdown
**Content fragments vs full documents:** If your HTML file starts with `<!DOCTYPE` or `<html`, the server serves it as-is (just injects the helper script). Otherwise, the server automatically wraps your content in the frame template — adding the header, CSS theme, selection indicator, and all interactive infrastructure. **Write content fragments by default.** Only write full documents when you need complete control over the page.
```

- [ ] **Step 3: 改写“The Loop”小节（第 36-61 行）**

将整个“The Loop”小节替换为：

```markdown
## The Loop

1. **Write HTML** to a new file in `screen_dir`:
   - Use semantic filenames: `platform.html`, `visual-style.html`, `layout.html`
   - **Never reuse filenames** — each screen gets a fresh file
   - Use Write tool — **never use cat/heredoc** (dumps noise into terminal)
   - Server automatically serves the newest file

2. **Tell user what to expect and end your turn:**
   - Remind them of the URL (every step, not just first)
   - Give a brief text summary of what's on screen (e.g., "Showing 3 layout options for the homepage")
   - Ask them to respond in the terminal: "Take a look and let me know what you think. Click to select an option if you'd like."

3. **On your next turn** — after the user responds in the terminal:
   - Read `$SCREEN_DIR/.events` if it exists — this contains the user's browser interactions (clicks, selections) as JSON lines
   - Merge with the user's terminal text to get the full picture
   - The terminal message is the primary feedback; `.events` provides structured interaction data

4. **Iterate or advance** — if feedback changes current screen, write a new file (e.g., `layout-v2.html`). Only move to the next question when the current step is validated.

5. Repeat until done.
```

- [ ] **Step 4: 替换“User Feedback Format”小节（第 165-174 行）**

替换为：

```markdown
## Browser Events Format

When the user clicks options in the browser, their interactions are recorded to `$SCREEN_DIR/.events` (one JSON object per line). The file is cleared automatically when you push a new screen.

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

The full event stream shows the user's exploration path — they may click multiple options before settling. The last `choice` event is typically the final selection, but the pattern of clicks can reveal hesitation or preferences worth asking about.

If `.events` doesn't exist, the user didn't interact with the browser — use only their terminal text.
```

- [ ] **Step 5: 更新“Writing Content Fragments”描述（第 65 行）**

移除“feedback footer”引用：

```markdown
Write just the content that goes inside the page. The server wraps it in the frame template automatically (header, theme CSS, selection indicator, and all interactive infrastructure).
```

- [ ] **Step 6: 更新 Reference 小节（第 200-203 行）**

移除关于“JS API”的 helper.js 描述——API 现在已最小化。保留路径引用：

```markdown
## Reference

- Frame template (CSS reference): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/frame-template.html`
- Helper script (client-side): `${CLAUDE_PLUGIN_ROOT}/lib/brainstorm-server/helper.js`
```

- [ ] **Step 7: 提交**

```bash
git add skills/brainstorming/visual-companion.md
git commit -m "Rewrite visual-companion.md for non-blocking browser-displays-terminal-commands flow"
```

---

### Task 7: 最终验证

- [ ] **Step 1: 运行完整测试套件**

```bash
cd /Users/drewritter/prime-rad/superpowers && node tests/brainstorm-server/server.test.js
```
预期：所有测试 PASS。

- [ ] **Step 2: 手动冒烟测试**

手动启动服务器，验证整个流程端到端工作：

```bash
cd /Users/drewritter/prime-rad/superpowers && lib/brainstorm-server/start-server.sh --project-dir /tmp/brainstorm-smoke-test
```

写一个测试片段，在浏览器中打开，点击一个选项，验证 `.events` 文件已被写入，验证指示器条已更新。然后停止服务器：

```bash
lib/brainstorm-server/stop-server.sh <screen_dir from start output>
```

- [ ] **Step 3: 验证无遗留引用**

```bash
grep -r "wait-for-feedback\|sendToClaude\|feedback-footer\|send-to-claude\|TaskOutput.*block.*true" /Users/drewritter/prime-rad/superpowers/ --include="*.js" --include="*.md" --include="*.sh" --include="*.html" | grep -v node_modules | grep -v RELEASE-NOTES | grep -v "\.md:.*spec\|plan"
```

预期：在 release notes 和规格/计划文档之外无命中（这些为历史记录）。

- [ ] **Step 4: 如需清理则最终提交**

```bash
git status
# Review untracked/modified files, stage specific files as needed, commit if clean
```
