# 根因追踪

## 概览

Bug 常常在调用栈深处显现（在错误目录 git init、文件建在错误位置、数据库用错误路径打开）。你的直觉是修错误出现的地方，但那是在治标。

**核心原则：** 沿调用链反向追踪，直到找到原始触发点，然后在源头修。

## 何时使用

```dot
digraph when_to_use {
    "Bug 在栈深处显现？" [shape=diamond];
    "能反向追踪？" [shape=diamond];
    "在症状处修" [shape=box];
    "追踪到原始触发点" [shape=box];
    "更好：同时加纵深防御" [shape=box];

    "Bug 在栈深处显现？" -> "能反向追踪？" [label="是"];
    "能反向追踪？" -> "追踪到原始触发点" [label="是"];
    "能反向追踪？" -> "在症状处修" [label="否 - 死胡同"];
    "追踪到原始触发点" -> "更好：同时加纵深防御";
}
```

**何时使用：**
- 错误发生在执行深处（不在入口点）
- 堆栈跟踪显示长调用链
- 不清楚无效数据源自哪里
- 需要找到哪个测试/代码触发了问题

## 追踪流程

### 1. 观察症状
```
Error: git init failed in ~/project/packages/core
```

### 2. 找到直接原因
**什么代码直接导致了这个？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：谁调用了它？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  ← 被 Session.initializeWorkspace() 调用
  ← 被 Session.create() 调用
  ← 被测试中的 Project.create() 调用
```

### 4. 持续向上追踪
**传入了什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 解析为 `process.cwd()`
- 那就是源代码目录！

### 5. 找到原始触发点
**空字符串从哪来？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前就访问了！
```

## 添加堆栈跟踪

当你无法手动追踪时，加埋点：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中用 `console.error()`（不要用 logger——可能不显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 找到哪个测试造成污染

如果某物在测试期间出现，但你不知道是哪个测试：

用本目录下的二分脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在第一个污染者处停下。用法见脚本。

## 真实示例：空的 projectDir

**症状：** `.git` 被建在 `packages/core/`（源代码）

**追踪链：**
1. `git init` 在 `process.cwd()` 中运行 ← 空的 cwd 参数
2. WorktreeManager 被传入空的 projectDir
3. Session.create() 传了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 初始返回 `{ tempDir: '' }`

**根因：** 顶层变量初始化访问了空值

**修复：** 把 tempDir 改成 getter，若在 beforeEach 之前访问就抛错

**同时加了纵深防御：**
- 第 1 层：Project.create() 校验目录
- 第 2 层：WorkspaceManager 校验非空
- 第 3 层：NODE_ENV 守卫拒绝在 tmpdir 外 git init
- 第 4 层：git init 前的堆栈跟踪日志

## 关键原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "能再向上追一层？" [shape=diamond];
    "反向追踪" [shape=box];
    "这是源头吗？" [shape=diamond];
    "在源头修" [shape=box];
    "在每层加校验" [shape=box];
    "Bug 不可能" [shape=doublecircle];
    "绝不只修症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "能再向上追一层？";
    "能再向上追一层？" -> "反向追踪" [label="是"];
    "能再向上追一层？" -> "绝不只修症状" [label="否"];
    "反向追踪" -> "这是源头吗？";
    "这是源头吗？" -> "反向追踪" [label="否 - 还在继续"];
    "这是源头吗？" -> "在源头修" [label="是"];
    "在源头修" -> "在每层加校验";
    "在每层加校验" -> "Bug 不可能";
}
```

**绝不在错误出现的地方就修。** 反向追踪找到原始触发点。

## 堆栈跟踪技巧

**在测试中：** 用 `console.error()` 而非 logger——logger 可能被抑制
**在操作之前：** 在危险操作之前记录，而非在它失败之后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整调用链

## 真实世界影响

来自调试会话（2025-10-03）：
- 通过 5 层追踪找到根因
- 在源头修复（getter 校验）
- 加了 4 层防御
- 1847 个测试通过，零污染
