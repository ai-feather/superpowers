# 可视化伴侣指南

基于浏览器的可视化头脑风暴伴侣，用于展示模型、图表和选项。

## 何时使用

逐问题决定，而非逐会话决定。判断标准是：**用户看到它会不会比读到它更容易理解？**

当内容本身是可视化的时**使用浏览器**：

- **UI 模型** —— 线框图、布局、导航结构、组件设计
- **架构图** —— 系统组件、数据流、关系图
- **并排可视化对比** —— 对比两种布局、两种配色方案、两种设计方向
- **设计打磨** —— 当问题关乎外观与感觉、间距、视觉层级时
- **空间关系** —— 以图表形式呈现的状态机、流程图、实体关系

当选择是非可视化的时候**使用 AskUserQuestion 工具**（它会渲染一个交互式选择器——绝不要把这些列为纯编号/字母文本）：

- **需求与范围问题** —— "X 是什么意思？"、"哪些功能在范围内？"
- **概念性 A/B/C 选择** —— 在用文字描述的方案之间挑选
- **权衡选择** —— 在利弊或对比选项之间选择
- **技术决策** —— API 设计、数据建模、架构方案选择
- **澄清问题** —— 任何答案是选择而非可视化偏好的情况

（纯终端文本只适用于真正开放式、答案是自由文本而非选择的问题。）

一个*关于* UI 话题的问题并不自动就是可视化问题。"你想要什么样的向导？"是概念性的——用 AskUserQuestion。"这些向导布局中哪种感觉对？"是可视化的——用浏览器。

## 工作原理

服务器监视一个目录中的 HTML 文件，并把最新的一个提供给浏览器。你把 HTML 内容写到 `screen_dir`，用户在浏览器中看到它，并可以点击选择选项。选择会被记录到 `state_dir/events`，你在下一轮读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器原样提供（只注入辅助脚本）。否则，服务器会自动把你的内容包裹进框架模板——加上页头、CSS 主题、连接状态以及全部交互基础设施。**默认写内容片段。** 只有当你需要对页面有完全控制时才写完整文档。

## 启动会话

```bash
# 在用户批准使用伴侣之后启动。--open 会在第一个屏幕自动打开用户的浏览器；
# --project-dir 会持久化模型，并启用同端口重启。
scripts/start-server.sh --project-dir /path/to/project --open

# 返回：{"type":"server-started","port":52341,
#           "url":"http://localhost:52341/?key=ab12…",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

从响应中保存 `screen_dir` 和 `state_dir`。使用 `--open` 时，当你推送第一个屏幕时浏览器会自行打开——你不需要请用户去打开它，但仍然要把 URL 作为后备分享出来（无头/远程环境不会自动打开）。

**URL 中包含一个会话密钥（`?key=…`）。** 服务器会拒绝任何不带它的请求，所以总是把 `url` 字段里的**完整** URL 给用户——绝不剥离查询字符串，也绝不给出一个光秃秃的 `http://host:port`。这个密钥同时管控 HTTP 和 WebSocket 访问，这样走丢的浏览器标签页或同一网络中的另一台机器就没法读取屏幕或注入事件。首次加载后浏览器会通过 cookie 记住这个密钥，所以刷新和 `/files/*` 资源无需再次携带它。

**查找连接信息：** 服务器会把启动 JSON 写到 `$STATE_DIR/server-info`。如果你是在后台启动服务器且没有捕获 stdout，就读那个文件来获取 URL 和端口。使用 `--project-dir` 时，到 `<project>/.superpowers/brainstorm/` 下找会话目录。

**注意：** 把项目根目录作为 `--project-dir` 传入，这样模型会持久化到 `.superpowers/brainstorm/` 并在服务器重启后保留。不传的话，文件会进 `/tmp` 并被清理。提醒用户如果 `.gitignore` 里还没有 `.superpowers/`，就把它加进去。

**按平台启动服务器：**

**Claude Code:**
```bash
# 默认模式即可——脚本自己会把服务器放到后台。
scripts/start-server.sh --project-dir /path/to/project --open
```

在 Windows 上，脚本会自动检测并切换到前台模式（这会阻塞工具调用）。在 Bash 工具调用上使用 `run_in_background: true`，让服务器在多个对话轮次间存活，然后在下一轮读取 `$STATE_DIR/server-info` 来获取 URL 和端口。

**Codex:**
```bash
# Codex 会回收后台进程。脚本会自动检测 CODEX_CI 并
# 切换到前台模式。正常运行即可——不需要额外 flag。
scripts/start-server.sh --project-dir /path/to/project --open
```

**Gemini CLI:**
```bash
# 使用 --foreground，并在你的 shell 工具调用上设置 is_background: true
# 让进程跨轮次存活
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**Copilot CLI:**
```bash
# 使用 --foreground，并通过 bash 工具以 mode: "async" 启动服务器，
# 让进程跨轮次存活。捕获返回的 shellId 以便日后需要时用
# read_bash / stop_bash 与它交互。
scripts/start-server.sh --project-dir /path/to/project --open --foreground
```

**其他环境：** 服务器必须在多个对话轮次之间持续在后台运行。如果你的环境会回收分离的进程，使用 `--foreground` 并用你所在平台的后台执行机制来启动命令。

如果 URL 从你的浏览器无法访问（在远程/容器化环境中很常见），绑定一个非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

用 `--url-host` 来控制返回的 URL JSON 中打印的主机名。

## 循环

1. **检查服务器是否还活着**，然后向 `screen_dir` 中的一个新文件**写入 HTML**：
   - **必须项：在引用 URL 或推送屏幕之前，确认服务器还活着。** 检查 `$STATE_DIR/server-info` 存在且 `$STATE_DIR/server-stopped` 不存在。如果它已关闭，用 `start-server.sh` 重启，使用**相同的 `--project-dir`**——它会复用同一个端口，所以用户已打开的标签页会自行重连（服务器关闭期间它显示一个"已暂停"的遮罩），你不需要发送新的 URL。服务器空闲 4 小时后会自动退出（可用 `--idle-timeout-minutes` 配置）。
   - 使用语义化的文件名：`platform.html`、`visual-style.html`、layout.html`
   - **绝不复用文件名** —— 每个屏幕都得到一个全新的文件
   - 使用你的文件创建工具——**绝不用 cat/heredoc**（会把噪声灌进终端）
   - 服务器自动提供最新的文件

2. **告诉用户预期什么，然后结束你的轮次：**
   - 提醒他们 URL（每一步都要，不只是第一次）
   - 给出屏幕上内容的简短文字摘要（例如"正在展示首页的 3 种布局方案"）
   - 请他们在终端里回复："看一下，告诉我你的想法。如果想选某个选项，点击选择即可。"

3. **在下一轮** —— 用户在终端回复之后：
   - 如果 `$STATE_DIR/events` 存在就读取它——这里面包含用户的浏览器交互（点击、选择），格式为 JSON 行
   - 把它与用户的终端文字合并，得到完整画面
   - 终端消息是主要反馈；`state_dir/events` 提供结构化的交互数据

4. **迭代或推进** —— 如果反馈改变了当前屏幕，写一个新文件（例如 `layout-v2.html`）。只有当前步骤被验证后才进入下一个问题。

5. **返回终端时卸载** —— 当下一步不需要浏览器时（例如一个澄清问题、一次权衡讨论），推送一个等待屏幕来清除陈旧内容：

   ```html
   <!-- 文件名：waiting.html（或 waiting-2.html 等） -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">Continuing in terminal...</p>
   </div>
   ```

   这能防止用户在对话已经推进时还盯着一个已确定的选项看。当下一个可视化问题出现时，照常推送一个新的内容文件。

6. 重复直到完成。

## 编写内容片段

只写要放进页面内部的内容。服务器会自动用框架模板把它包裹起来（页头、主题 CSS、连接状态以及全部交互基础设施）。

**最小示例：**

```html
<h2>Which layout works better?</h2>
<p class="subtitle">Consider readability and visual hierarchy</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Single Column</h3>
      <p>Clean, focused reading experience</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>Two Column</h3>
      <p>Sidebar navigation with main content</p>
    </div>
  </div>
</div>
```

这样就够了。不需要 `<html>`、不需要 CSS、不需要 `<script>` 标签。服务器会提供这一切。

## 可用的 CSS 类

框架模板为你的内容提供这些 CSS 类：

### Options（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>Title</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

**多选：** 给容器加上 `data-multiselect`，让用户可以选择多个选项。每次点击会切换该项目的选中样式。

```html
<div class="options" data-multiselect>
  <!-- 相同的 option 标记——用户可以选中/取消多个 -->
</div>
```

### Cards（可视化设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- mockup content --></div>
    <div class="card-body">
      <h3>Name</h3>
      <p>Description</p>
    </div>
  </div>
</div>
```

### Mockup 容器

```html
<div class="mockup">
  <div class="mockup-header">Preview: Dashboard Layout</div>
  <div class="mockup-body"><!-- your mockup HTML --></div>
</div>
```

### Split view（并排）

```html
<div class="split">
  <div class="mockup"><!-- left --></div>
  <div class="mockup"><!-- right --></div>
</div>
```

### Pros/Cons

```html
<div class="pros-cons">
  <div class="pros"><h4>Pros</h4><ul><li>Benefit</li></ul></div>
  <div class="cons"><h4>Cons</h4><ul><li>Drawback</li></ul></div>
</div>
```

### Mock 元素（线框图构件）

```html
<div class="mock-nav">Logo | Home | About | Contact</div>
<div style="display: flex;">
  <div class="mock-sidebar">Navigation</div>
  <div class="mock-content">Main content area</div>
</div>
<button class="mock-button">Action Button</button>
<input class="mock-input" placeholder="Input field">
<div class="placeholder">Placeholder area</div>
```

### 排版与章节

- `h2` —— 页面标题
- `h3` —— 章节标题
- `.subtitle` —— 标题下方的次要文字
- `.section` —— 带下边距的内容块
- `.label` —— 小号大写标签文字

## 浏览器事件格式

当用户在浏览器中点击选项时，他们的交互会被记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时，这个文件会自动清空。

```jsonl
{"type":"click","choice":"a","text":"Option A - Simple Layout","timestamp":1706000101}
{"type":"click","choice":"c","text":"Option C - Complex Grid","timestamp":1706000108}
{"type":"click","choice":"b","text":"Option B - Hybrid","timestamp":1706000115}
```

完整的事件流展示了用户的探索路径——他们可能在定下来之前点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击的模式可能揭示值得追问的犹豫或偏好。

如果 `$STATE_DIR/events` 不存在，说明用户没有在浏览器里交互——只用他们的终端文字。

## 设计技巧

- **保真度与问题相称** —— 布局问题用线框图，打磨问题用精细稿
- **在每一页解释清楚问题** —— "哪种布局看起来更专业？"而不只是"选一个"
- **推进前先迭代** —— 如果反馈改变了当前屏幕，写一个新版本
- **每屏最多 2-4 个选项**
- **在重要的地方使用真实内容** —— 比如摄影作品集，用真实图片（Unsplash）。占位内容会掩盖设计问题。
- **保持模型简单** —— 聚焦于布局和结构，而非像素级完美的设计

## 文件命名

- 使用语义化名称：`platform.html`、`visual-style.html`、`layout.html`
- 绝不复用文件名——每个屏幕必须是一个新文件
- 迭代时：加上版本后缀，如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新的文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，模型文件会保留在 `.superpowers/brainstorm/` 中供日后参考。只有 `/tmp` 会话在停止时会被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
