# 可视化头脑风暴重构：浏览器负责展示，终端负责对话

**日期：** 2026-02-19
**状态：** Approved
**范围：** `lib/brainstorm-server/`、`skills/brainstorming/visual-companion.md`、`tests/brainstorm-server/`

## 问题

在可视化头脑风暴过程中，Claude 把 `wait-for-feedback.sh` 作为后台任务运行，并阻塞在 `TaskOutput(block=true, timeout=600s)` 上。这会完全占用 TUI——用户在可视化头脑风暴运行期间无法向 Claude 输入。浏览器成为唯一的输入通道。

Claude Code 的执行模型是基于回合的（turn-based）。在单个回合内，Claude 无法同时监听两个通道。阻塞式 `TaskOutput` 是错误的原语——它模拟了平台并不支持的事件驱动行为。

## 设计

### 核心模型

**浏览器 = 交互式展示。** 显示 mockup，让用户点击选择选项。选择结果在服务端记录。

**终端 = 对话通道。** 始终不阻塞，始终可用。用户在这里与 Claude 对话。

### 循环流程

1. Claude 将一个 HTML 文件写入会话目录
2. 服务器通过 chokidar 检测到它，通过 WebSocket 推送 reload 给浏览器（不变）
3. Claude 结束回合——告诉用户去查看浏览器并在终端中回应
4. 用户查看浏览器，可选地点击选择一个选项，然后在终端中输入反馈
5. 在下一个回合中，Claude 读取 `$SCREEN_DIR/.events` 获取浏览器交互流（点击、选择），并与终端文本合并
6. 迭代或推进

没有后台任务。没有 `TaskOutput` 阻塞。没有轮询脚本。

### 关键删除：`wait-for-feedback.sh`

完全删除。它的目的是桥接“服务器把事件日志写到 stdout”和“Claude 需要接收这些事件”。`.events` 文件取代了它——服务器直接写入用户交互事件，Claude 用平台提供的任何文件读取机制来读取它们。

### 关键新增：`.events` 文件（每 screen 事件流）

服务器把所有用户交互事件写入 `$SCREEN_DIR/.events`，每行一个 JSON 对象。这给 Claude 提供了当前 screen 的完整交互流——不只是最终选择，还有用户的探索路径（点了 A，然后 B，最后定在 C）。

用户探索选项后的示例内容：

```jsonl
{"type":"click","choice":"a","text":"Option A - Preset-First Wizard","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Manual Config","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid Approach","timestamp":1706000115}
```

- 在一个 screen 内是 append-only（仅追加）。每个用户事件作为新行追加。
- 当 chokidar 检测到新 HTML 文件（推送新 screen）时，文件被清空（删除），防止陈旧事件残留。
- 如果 Claude 读取时文件不存在，说明没有发生浏览器交互——Claude 只使用终端文本。
- 文件只包含用户事件（`click` 等）——不包含服务器生命周期事件（`server-started`、`screen-added`）。这让它保持小而聚焦。
- Claude 可以读取完整流来理解用户的探索模式，或者只看最后一个 `choice` 事件来获取最终选择。

## 按文件列出改动

### `index.js`（服务器）

**A. 把用户事件写入 `.events` 文件。**

在 WebSocket 的 `message` handler 中，在把事件日志写到 stdout 之后：通过 `fs.appendFileSync` 把该事件作为 JSON 行追加到 `$SCREEN_DIR/.events`。只写入用户交互事件（带 `source: 'user-event'` 的那些），不写服务器生命周期事件。

**B. 在新 screen 时清空 `.events`。**

在 chokidar 的 `add` handler（检测到新 `.html` 文件）中，如果 `$SCREEN_DIR/.events` 存在则删除它。这是明确的“新 screen”信号——比在 GET `/` 时清空更好，因为后者在每次 reload 时都会触发。

**C. 替换 `wrapInFrame` 的内容注入。**

当前的正则锚定在 `<div class="feedback-footer">` 上，而后者将被移除。替换为一个注释占位符：移除 `#claude-content` 内现有的默认内容（`<h2>Visual Brainstorming</h2>` 和副标题段落），替换为一个单独的 `<!-- CONTENT -->` 标记。内容注入变成 `frameTemplate.replace('<!-- CONTENT -->', content)`。更简单，也不会因为模板格式变化而破坏。

### `frame-template.html`（UI 框架）

**移除：**
- `feedback-footer` div（textarea、Send 按钮、label、`.feedback-row`）
- 相关 CSS（`.feedback-footer`、`.feedback-footer label`、`.feedback-row`，以及其中的 textarea 和 button 样式）

**新增：**
- 在 `#claude-content` 内的 `<!-- CONTENT -->` 占位符，替换掉默认文本
- 在 footer 原位置增加一个选择指示条，有两个状态：
  - 默认：“Click an option above, then return to the terminal”
  - 选择后：“Option B selected — return to terminal to continue”
- 指示条的 CSS（低调，视觉权重与现有 header 相当）

**保持不变：**
- 带有 “Brainstorm Companion” 标题和连接状态的 header 条
- `.main` 包装器和 `#claude-content` 容器
- 所有组件 CSS（`.options`、`.cards`、`.mockup`、`.split`、`.pros-cons`、placeholder、mock 元素）
- 深色/浅色主题变量和 media query

### `helper.js`（客户端脚本）

**移除：**
- `sendToClaude()` 函数以及 “Sent to Claude” 页面接管
- `window.send()` 函数（与已移除的 Send 按钮绑定）
- 表单提交 handler——没有 feedback textarea 后没有意义，还增加日志噪声
- 输入变化 handler——同理
- `pageshow` 事件监听器（当初为修复 textarea 持久化而加——已经没有 textarea 了）

**保留：**
- WebSocket 连接、重连逻辑、事件队列
- Reload handler（服务器推送时 `window.location.reload()`）
- `window.toggleSelect()` 用于选择高亮
- `window.selectedChoice` 追踪
- `window.brainstorm.send()` 和 `window.brainstorm.choice()`——它们与已移除的 `window.send()` 不同。它们调用 `sendEvent`，后者通过 WebSocket 把日志发给服务器。对自定义的整页文档页面有用。

**收窄：**
- Click handler：只捕获 `[data-choice]` 的点击，而不是所有 button/link。当浏览器作为反馈通道时需要宽捕获；现在只用于选择追踪。

**新增：**
- 在 `data-choice` 点击时，更新选择指示条文字以显示选中了哪个选项。

**从 `window.brainstorm` API 中移除：**
- `brainstorm.sendToClaude`——已不存在

### `visual-companion.md`（技能指令）

**重写 “The Loop” 章节** 为上述的非阻塞流程。移除所有对以下内容的引用：
- `wait-for-feedback.sh`
- `TaskOutput` 阻塞
- 超时/重试逻辑（600s 超时、30 分钟上限）
- 描述 `send-to-claude` JSON 的 “User Feedback Format” 章节

**替换为：**
- 新的循环流程（写 HTML → 结束回合 → 用户在终端回应 → 读 `.events` → 迭代）
- `.events` 文件格式文档
- 指导说明：终端消息是主要反馈；`.events` 提供完整浏览器交互流作为补充上下文

**保留：**
- 服务器启动/关闭指令
- 内容片段 vs 整篇文档的指导
- CSS class 参考和可用组件
- 设计提示（保真度与问题相匹配、每 screen 2-4 个选项等）

### `wait-for-feedback.sh`

**完全删除。**

### `tests/brainstorm-server/server.test.js`

需要更新的测试：
- 断言片段响应中存在 `feedback-footer` 的测试——更新为断言选择指示条或 `<!-- CONTENT -->` 替换
- 断言 `helper.js` 包含 `send` 的测试——更新以反映收窄后的 API
- 断言 `sendToClaude` CSS 变量使用的测试——移除（函数已不存在）

## 平台兼容性

服务器代码（`index.js`、`helper.js`、`frame-template.html`）完全平台无关——纯 Node.js 和浏览器 JavaScript。没有任何 Claude Code 特定的引用。已经证明可以通过后台终端交互在 Codex 上运行。

技能指令（`visual-companion.md`）是平台适配层。每个平台的 Claude 用自己的工具来启动服务器、读取 `.events` 等。非阻塞模型天然跨平台工作，因为它不依赖任何平台特定的阻塞原语。

## 此设计带来的好处

- **TUI 始终响应**，在可视化头脑风暴期间
- **混合输入** —— 在浏览器点击 + 在终端输入，自然合并
- **优雅降级** —— 浏览器挂了或用户没打开？终端仍然可用
- **更简单的架构** —— 没有后台任务、没有轮询脚本、没有超时管理
- **跨平台** —— 同样的服务器代码可在 Claude Code、Codex 以及任何未来的平台上工作

## 此设计放弃的东西

- **纯浏览器反馈工作流** —— 用户必须返回终端才能继续。选择指示条会引导他们，但相比旧的“点击 Send 然后等待”流程多了一步。
- **来自浏览器的内联文本反馈** —— textarea 没了。所有文本反馈都走终端。这是有意为之——终端是比 frame 中小 textarea 更好的文本输入通道。
- **浏览器 Send 后的即时响应** —— 旧系统在用户点击 Send 时立即让 Claude 响应。现在用户切回终端时有一段间隙。实际中这是几秒，并且用户可以在终端消息中补充上下文。
