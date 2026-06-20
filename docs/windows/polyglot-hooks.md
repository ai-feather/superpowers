# Claude Code 的跨平台多语言 Hooks

Claude Code 插件需要能在 Windows、macOS 和 Linux 上都正常工作的 hooks。本文档介绍 `hooks/run-hook.cmd` 中使用的单一通用分发器模式。

> **权威来源：** `hooks/run-hook.cmd` 是规范实现。当本文档与代码不一致时，以代码为准。

## 问题所在

Claude Code 通过系统默认 shell 来运行 hook 命令：
- **Windows**：CMD.exe
- **macOS/Linux**：bash 或 sh

这带来了几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：`$VAR` 语法在 CMD 中不工作
4. **`.sh` 自动前置**：Windows 上的 Claude Code 会自动为任何路径中包含 `.sh` 的命令前置 `bash` —— 当脚本带扩展名时会干扰分发器

## 解决方案：无扩展名脚本 + 单一通用分发器

本仓库对所有 hooks 使用同一个通用的 `run-hook.cmd` 分发器。Hook 脚本是**无扩展名的**（`session-start`，而不是 `session-start.sh`）。这是有意为之：它避免 Claude Code 在 Windows 上的自动检测机制给分发器命令前置 `bash`，从而避免破坏分发器。

### 文件结构

```
hooks/
├── hooks.json          # 指向 run-hook.cmd 并使用无扩展名脚本名
├── run-hook.cmd        # 跨平台分发器（多语言包装器）
└── session-start       # 真正的 hook 逻辑 —— 无扩展名的 bash 脚本
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ]
  }
}
```

路径之所以加引号，是因为 `${CLAUDE_PLUGIN_ROOT}` 可能包含空格。

## `run-hook.cmd` 的高层工作原理

`run-hook.cmd` 是一个多语言脚本：Windows 把第一个代码块当作 batch 命令执行，而 Unix shell 则把该代码块当作 no-op heredoc 略过，并继续执行其后的内容。

不要从本文档拷贝实现。修改分发器时请直接阅读 `hooks/run-hook.cmd`，并在之后运行 `tests/hooks/test-session-start.sh`。

### 在 Windows（CMD.exe）上如何工作

1. batch 部分会校验脚本名，并根据分发器自身所在位置解析出 hook 目录。
2. 它会在三处尝试 bash：
   - `C:\Program Files\Git\bin\bash.exe`
   - `C:\Program Files (x86)\Git\bin\bash.exe`
   - `PATH` 中的 `bash`（MSYS2、Cygwin 或非默认安装的 Git）
3. 如果找到 bash，就从 hooks 目录运行具名的无扩展名 hook 脚本。
4. 如果找不到 bash，分发器会静默地以 `0` 退出 —— 插件继续工作，只是跳过该 hook。
5. `exit /b` 会在 CMD 进入 Unix 部分之前停止执行。

### 在 Unix（bash/sh）上如何工作

1. `: << 'CMDBLOCK'` 在一个 no-op 命令上开启一个 heredoc。
2. 整个 CMD batch 代码块都被 heredoc 消费并忽略。
3. 在 `CMDBLOCK` 之后，bash 解析脚本目录并直接 `exec` 具名的无扩展名脚本。

### 关键设计决策

| 决策 | 原因 |
|----------|-----|
| 无扩展名脚本 | 避免 Claude Code 在 Windows 上的 `.sh` 自动前置干扰分发器命令 |
| 不使用 `-l`（登录 shell） | 不需要；hook 脚本应当自包含，不依赖登录 shell 的 PATH 设置 |
| 不使用 `cygpath` | Bash 直接接收 Windows 路径并能正确处理；旧的 `-c "..."` 调用模式才需要 `cygpath`，直接 exec 时不需要 |
| 无 bash 时静默退出 | 避免对未安装 Git for Windows 的用户造成插件破坏；hook 上下文注入会被优雅地跳过 |

## 编写跨平台 Hook 脚本

你的 hook 逻辑放在无扩展名的脚本文件中。以下是一些可移植的写法：

### 应当

- 尽可能使用纯 bash 内建命令
- 使用 `$(command)` 而非反引号
- 对所有变量展开加引号：`"$VAR"`

### 避免

- 在没有回退方案的情况下依赖 PATH 相关的工具（hook 在不使用 `-l` 时运行，因此登录 shell 的 PATH 未被设置）
- 给脚本加 `.sh` 扩展名 —— 这会触发 Claude Code 在 Windows 上的自动前置

### 示例：不依赖外部工具的 JSON 转义

```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 故障排查

### "bash is not recognized"

CMD 在分发器尝试的三个位置都没能找到 bash。分发器会静默退出（0）而不是报错，因此该 hook 会被跳过。请按标准路径安装 Git for Windows，或者确保 `bash` 在 `PATH` 中。

### Hook 在 Unix 上正常运行但在 Windows 上什么都不做

请检查 `hooks.json` 中的脚本文件名是否**无扩展名**。形如 `run-hook.cmd session-start.sh` 的命令可能触发 Claude Code 的 `.sh` 自动检测，从而绕过预期的 CMD 分发器路径，或者只是去尝试运行一个并不存在的 `session-start.sh` 脚本。

### Hook 完全不触发

请核对 `hooks.json` 中的 `matcher` 是否与你宿主发出的事件类型相匹配。Claude Code 使用 `startup|clear|compact`；Codex 使用 `startup|resume|clear`。Codex 的变体请查看 `hooks-codex.json`。

## 相关 Issue

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) —— `.sh` 脚本在 Windows 上被编辑器打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) —— Hooks 在 Windows 上不工作
