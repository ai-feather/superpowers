# 来自用户反馈的技能改进

**日期：** 2025-11-28
**状态：** 草稿
**来源：** 两个在真实开发场景中使用 superpowers 的 Claude 实例

---

## 执行摘要

两个 Claude 实例从实际开发会话中提供了详细反馈。他们的反馈揭示了当前技能中存在的**系统性缺口**，这些缺口导致即便遵循了技能，本可避免的 bug 仍然被发布出去。

**关键洞察：** 这些是问题报告，而不仅是解决方案提案。问题是真实的；解决方案需要仔细评估。

**关键主题：**
1. **验证缺口** - 我们验证操作成功，却没有验证它们达成了预期结果
2. **进程卫生** - 后台进程会累积并在不同子代理之间相互干扰
3. **上下文优化** - 子代理收到过多无关信息
4. **缺失自省** - 在交接前没有提示去审视自己的工作
5. **Mock 安全** - Mock 可能偏离接口而不被发现
6. **技能激活** - 技能存在但未被阅读/使用

---

## 已识别的问题

### 问题 1：配置变更验证缺口

**发生了什么：**
- 子代理测试"OpenAI 集成"
- 设置了 `OPENAI_API_KEY` 环境变量
- 收到了状态 200 响应
- 报告"OpenAI 集成工作正常"
- **但是**响应中包含 `"model": "claude-sonnet-4-20250514"` —— 实际上使用的是 Anthropic

**根因：**
`verification-before-completion` 检查操作是否成功，但没有检查结果是否反映了预期的配置变更。

**影响：** 高 —— 对集成测试产生虚假信心，bug 被发布到生产环境

**典型失败模式：**
- 切换 LLM 提供商 → 验证状态 200 但不检查模型名称
- 启用功能开关 → 验证没有错误但不检查功能是否激活
- 更改环境 → 验证部署成功但不检查环境变量

---

### 问题 2：后台进程累积

**发生了什么：**
- 会话期间派发了多个子代理
- 每个都启动了后台服务器进程
- 进程不断累积（4 个以上服务器在运行）
- 僵尸进程仍占用端口
- 后来的 E2E 测试命中了配置错误的僵尸服务器
- 产生令人困惑/错误的测试结果

**根因：**
子代理是无状态的 —— 它们不知道前序子代理的进程。没有清理协议。

**影响：** 中高 —— 测试命中错误的服务器、产生错误的通过/失败、调试困难

---

### 问题 3：子代理提示词中的上下文臃肿

**发生了什么：**
- 标准做法：让子代理读取完整的计划文件
- 实验：只给任务 + 模式 + 文件 + 验证命令
- 结果：更快、更专注，单次尝试即完成更常见

**根因：**
子代理在无关的计划章节上浪费 token 和注意力。

**影响：** 中 —— 执行更慢，更多失败尝试

**有效的做法：**
```
You are adding a single E2E test to packnplay's test suite.

**Your task:** Add `TestE2E_FeaturePrivilegedMode` to `pkg/runner/e2e_test.go`

**What to test:** A local devcontainer feature that requests `"privileged": true`
in its metadata should result in the container running with `--privileged` flag.

**Follow the exact pattern of TestE2E_FeatureOptionValidation** (at the end of the file)

**After writing, run:** `go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m`
```

---

### 问题 4：交接前缺失自省

**发生了什么：**
- 添加了自省提示词："以全新的眼光审视你的工作 —— 还有什么可以改进？"
- 任务 5 的实现者发现失败的测试是由实现 bug 引起的，而非测试 bug
- 追溯到第 99 行：`strings.Join(metadata.Entrypoint, " ")` 创建了无效的 Docker 语法
- 如果没有自省，只会报告"测试失败"而没有根因

**根因：**
实现者在报告完成之前不会自然地退一步审视自己的工作。

**影响：** 中 —— 实现者本可发现的 bug 被交接给了审查者

---

### 问题 5：Mock 与接口的偏离

**发生了什么：**
```typescript
// Interface defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}

// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) defines cleanup()
vi.mock('web-adapter', () => ({
  WebAdapter: vi.fn().mockImplementation(() => ({
    cleanup: vi.fn().mockResolvedValue(undefined),  // Wrong!
  })),
}));
```
- 测试通过
- 运行时崩溃："adapter.cleanup is not a function"

**根因：**
Mock 源自 buggy 代码的调用，而非接口定义。TypeScript 无法捕获方法名错误的内联 mock。

**影响：** 高 —— 测试给出虚假信心，运行时崩溃

**为什么 testing-anti-patterns 没能阻止这种情况：**
该技能涵盖了测试 mock 行为和不理解就 mock 的做法，但没有涵盖"从接口派生 mock，而非实现"这一特定模式。

---

### 问题 6：代码审查者的文件访问问题

**发生了什么：**
- 派发了代码审查者子代理
- 找不到测试文件："该文件在仓库中似乎不存在"
- 文件实际存在
- 审查者不知道应先显式读取它

**根因：**
审查者提示词中没有包含显式的文件读取指令。

**影响：** 低到中 —— 审查失败或不完整

---

### 问题 7：修复工作流的延迟

**发生了什么：**
- 实现者在自省时发现 bug
- 实现者知道修复方法
- 当前工作流：报告 → 我派发修复者 → 修复者修复 → 我验证
- 额外的往返增加了延迟，却没增加价值

**根因：**
当实现者已完成诊断时，实现者和修复者角色之间的僵化分离。

**影响：** 低 —— 存在延迟，但没有正确性问题

---

### 问题 8：技能未被阅读

**发生了什么：**
- `testing-anti-patterns` 技能存在
- 无论是人类还是子代理，在编写测试前都没阅读它
- 原本可以避免一些问题（虽然不是全部 —— 见问题 5）

**根因：**
没有强制要求子代理阅读相关技能。没有提示词包含阅读技能的环节。

**影响：** 中 —— 如果不被使用，技能投入就被浪费了

---

## 建议的改进

### 1. verification-before-completion：添加配置变更验证

**新增章节：**

```markdown
## Verifying Configuration Changes

When testing changes to configuration, providers, feature flags, or environment:

**Don't just verify the operation succeeded. Verify the output reflects the intended change.**

### Common Failure Pattern

Operation succeeds because *some* valid config exists, but it's not the config you intended to test.

### Examples

| Change | Insufficient | Required |
|--------|-------------|----------|
| Switch LLM provider | Status 200 | Response contains expected model name |
| Enable feature flag | No errors | Feature behavior actually active |
| Change environment | Deploy succeeds | Logs/vars reference new environment |
| Set credentials | Auth succeeds | Authenticated user/context is correct |

### Gate Function

```
BEFORE claiming configuration change works:

1. IDENTIFY: What should be DIFFERENT after this change?
2. LOCATE: Where is that difference observable?
   - Response field (model name, user ID)
   - Log line (environment, provider)
   - Behavior (feature active/inactive)
3. RUN: Command that shows the observable difference
4. VERIFY: Output contains expected difference
5. ONLY THEN: Claim configuration change works

Red flags:
  - "Request succeeded" without checking content
  - Checking status code but not response body
  - Verifying no errors but not positive confirmation
```

**Why this works:**
Forces verification of INTENT, not just operation success.

---

### 2. subagent-driven-development：为 E2E 测试添加进程卫生

**新增章节：**

```markdown
## Process Hygiene for E2E Tests

When dispatching subagents that start services (servers, databases, message queues):

### Problem

Subagents are stateless - they don't know about processes started by previous subagents. Background processes persist and can interfere with later tests.

### Solution

**Before dispatching E2E test subagent, include cleanup in prompt:**

```
BEFORE starting any services:
1. Kill existing processes: pkill -f "<service-pattern>" 2>/dev/null || true
2. Wait for cleanup: sleep 1
3. Verify port free: lsof -i :<port> && echo "ERROR: Port still in use" || echo "Port free"

AFTER tests complete:
1. Kill the process you started
2. Verify cleanup: pgrep -f "<service-pattern>" || echo "Cleanup successful"
```

### Example

```
Task: Run E2E test of API server

Prompt includes:
"Before starting the server:
- Kill any existing servers: pkill -f 'node.*server.js' 2>/dev/null || true
- Verify port 3001 is free: lsof -i :3001 && exit 1 || echo 'Port available'

After tests:
- Kill the server you started
- Verify: pgrep -f 'node.*server.js' || echo 'Cleanup verified'"
```

### Why This Matters

- Stale processes serve requests with wrong config
- Port conflicts cause silent failures
- Process accumulation slows system
- Confusing test results (hitting wrong server)
```

**权衡分析：**
- 增加了提示词样板
- 但避免了非常令人困惑的调试
- 对于 E2E 测试子代理是值得的

---

### 3. subagent-driven-development：添加精简上下文选项

**修改步骤 2：使用子代理执行任务**

**之前：**
```
Read that task carefully from [plan-file].
```

**之后：**
```
## Context Approaches

**Full Plan (default):**
Use when tasks are complex or have dependencies:
```
Read Task N from [plan-file] carefully.
```

**Lean Context (for independent tasks):**
Use when task is standalone and pattern-based:
```
You are implementing: [1-2 sentence task description]

File to modify: [exact path]
Pattern to follow: [reference to existing function/test]
What to implement: [specific requirement]
Verification: [exact command to run]

[Do NOT include full plan file]
```

**Use lean context when:**
- Task follows existing pattern (add similar test, implement similar feature)
- Task is self-contained (doesn't need context from other tasks)
- Pattern reference is sufficient (e.g., "follow TestE2E_FeatureOptionValidation")

**Use full plan when:**
- Task has dependencies on other tasks
- Requires understanding of overall architecture
- Complex logic that needs context
```

**示例：**
```
Lean context prompt:

"You are adding a test for privileged mode in devcontainer features.

File: pkg/runner/e2e_test.go
Pattern: Follow TestE2E_FeatureOptionValidation (at end of file)
Test: Feature with `"privileged": true` in metadata results in `--privileged` flag
Verify: go test -v ./pkg/runner -run TestE2E_FeaturePrivilegedMode -timeout 5m

Report: Implementation, test results, any issues."
```

**为何有效：**
在适用时减少 token 使用，提升专注度，更快完成。

---

### 4. subagent-driven-development：添加自省步骤

**修改步骤 2：使用子代理执行任务**

**添加到提示词模板：**

```
When done, BEFORE reporting back:

Take a step back and review your work with fresh eyes.

Ask yourself:
- Does this actually solve the task as specified?
- Are there edge cases I didn't consider?
- Did I follow the pattern correctly?
- If tests are failing, what's the ROOT CAUSE (implementation bug vs test bug)?
- What could be better about this implementation?

If you identify issues during this reflection, fix them now.

Then report:
- What you implemented
- Self-reflection findings (if any)
- Test results
- Files changed
```

**为何有效：**
在交接前捕获实现者自己能发现的 bug。有记录的案例：通过自省识别出 entrypoint bug。

**权衡：**
每个任务增加约 30 秒，但在审查前就能发现问题。

---

### 5. requesting-code-review：添加显式文件读取

**修改 code-reviewer 模板：**

**在开头添加：**

```markdown
## Files to Review

BEFORE analyzing, read these files:

1. [List specific files that changed in the diff]
2. [Files referenced by changes but not modified]

Use Read tool to load each file.

If you cannot find a file:
- Check exact path from diff
- Try alternate locations
- Report: "Cannot locate [path] - please verify file exists"

DO NOT proceed with review until you've read the actual code.
```

**为何有效：**
显式指令可以防止"文件未找到"的问题。

---

### 6. testing-anti-patterns：添加 Mock 与接口偏离的反模式

**新增反模式 6：**

```markdown
## Anti-Pattern 6: Mocks Derived from Implementation

**The violation:**
```typescript
// Code (BUGGY) calls cleanup()
await adapter.cleanup();

// Mock (MATCHES BUG) has cleanup()
const mock = {
  cleanup: vi.fn().mockResolvedValue(undefined)
};

// Interface (CORRECT) defines close()
interface PlatformAdapter {
  close(): Promise<void>;
}
```

**Why this is wrong:**
- Mock encodes the bug into the test
- TypeScript can't catch inline mocks with wrong method names
- Test passes because both code and mock are wrong
- Runtime crashes when real object is used

**The fix:**
```typescript
// ✅ GOOD: Derive mock from interface

// Step 1: Open interface definition (PlatformAdapter)
// Step 2: List methods defined there (close, initialize, etc.)
// Step 3: Mock EXACTLY those methods

const mock = {
  initialize: vi.fn().mockResolvedValue(undefined),
  close: vi.fn().mockResolvedValue(undefined),  // From interface!
};

// Now test FAILS because code calls cleanup() which doesn't exist
// That failure reveals the bug BEFORE runtime
```

### Gate Function

```
BEFORE writing any mock:

  1. STOP - Do NOT look at the code under test yet
  2. FIND: The interface/type definition for the dependency
  3. READ: The interface file
  4. LIST: Methods defined in the interface
  5. MOCK: ONLY those methods with EXACTLY those names
  6. DO NOT: Look at what your code calls

  IF your test fails because code calls something not in mock:
    ✅ GOOD - The test found a bug in your code
    Fix the code to call the correct interface method
    NOT the mock

  Red flags:
    - "I'll mock what the code calls"
    - Copying method names from implementation
    - Mock written without reading interface
    - "The test is failing so I'll add this method to the mock"
```

**Detection:**

When you see runtime error "X is not a function" and tests pass:
1. Check if X is mocked
2. Compare mock methods to interface methods
3. Look for method name mismatches
```

**为何有效：**
直接针对反馈中的失败模式。

---

### 7. subagent-driven-development：要求测试子代理阅读技能

**当任务涉及测试时，添加到提示词模板：**

```markdown
BEFORE writing any tests:

1. Read testing-anti-patterns skill:
   Use Skill tool: superpowers:testing-anti-patterns

2. Apply gate functions from that skill when:
   - Writing mocks
   - Adding methods to production classes
   - Mocking dependencies

This is NOT optional. Tests that violate anti-patterns will be rejected in review.
```

**为何有效：**
确保技能真正被使用，而不仅仅是存在。

**权衡：**
每个任务增加时间，但能防止整类 bug。

---

### 8. subagent-driven-development：允许实现者修复自己发现的问题

**修改步骤 2：**

**当前：**
```
Subagent reports back with summary of work.
```

**建议：**
```
Subagent performs self-reflection, then:

IF self-reflection identifies fixable issues:
  1. Fix the issues
  2. Re-run verification
  3. Report: "Initial implementation + self-reflection fix"

ELSE:
  Report: "Implementation complete"

Include in report:
- Self-reflection findings
- Whether fixes were applied
- Final verification results
```

**为何有效：**
当实现者已经知道修复方法时，减少延迟。有记录的案例：本可节省 entrypoint bug 的一次往返。

**权衡：**
提示词略复杂，但端到端更快。

---

## 实现计划

### 阶段 1：高影响、低风险（优先实施）

1. **verification-before-completion：配置变更验证**
   - 清晰的追加，不改变现有内容
   - 解决高影响问题（对测试的虚假信心）
   - 文件：`skills/verification-before-completion/SKILL.md`

2. **testing-anti-patterns：Mock 与接口偏离**
   - 添加新反模式，不修改现有内容
   - 解决高影响问题（运行时崩溃）
   - 文件：`skills/testing-anti-patterns/SKILL.md`

3. **requesting-code-review：显式文件读取**
   - 对模板的简单追加
   - 修复具体问题（审查者找不到文件）
   - 文件：`skills/requesting-code-review/SKILL.md`

### 阶段 2：中等变更（谨慎测试）

4. **subagent-driven-development：进程卫生**
   - 添加新章节，不改变工作流
   - 解决中高影响问题（测试可靠性）
   - 文件：`skills/subagent-driven-development/SKILL.md`

5. **subagent-driven-development：自省**
   - 改变提示词模板（风险较高）
   - 但有记录表明能捕获 bug
   - 文件：`skills/subagent-driven-development/SKILL.md`

6. **subagent-driven-development：技能阅读要求**
   - 增加提示词开销
   - 但确保技能被真正使用
   - 文件：`skills/subagent-driven-development/SKILL.md`

### 阶段 3：优化（先验证）

7. **subagent-driven-development：精简上下文选项**
   - 增加复杂性（两种方法）
   - 需要验证它不会导致混淆
   - 文件：`skills/subagent-driven-development/SKILL.md`

8. **subagent-driven-development：允许实现者修复**
   - 改变工作流（风险较高）
   - 属于优化，而非 bug 修复
   - 文件：`skills/subagent-driven-development/SKILL.md`

---

## 待解决问题

1. **精简上下文方式：**
   - 是否应将其作为基于模式的任务的默认选项？
   - 如何决定使用哪种方式？
   - 过于精简而遗漏重要上下文的风险？

2. **自省：**
   - 是否会显著拖慢简单任务？
   - 是否只应应用于复杂任务？
   - 如何防止"反思疲劳"，即变成例行公事？

3. **进程卫生：**
   - 应该放在 subagent-driven-development 还是单独的技能中？
   - 是否适用于 E2E 测试之外的其他工作流？
   - 如何处理进程本应持续存在的情况（开发服务器）？

4. **技能阅读强制：**
   - 是否应要求所有子代理阅读相关技能？
   - 如何避免提示词变得过长？
   - 过度文档化而失去专注的风险？

---

## 成功指标

我们如何知道这些改进有效？

1. **配置验证：**
   - "测试通过但使用了错误配置"的情况为零
   - Jesse 不再说"那其实没在测试你以为的东西"

2. **进程卫生：**
   - "测试命中了错误服务器"的情况为零
   - E2E 测试运行期间没有端口冲突错误

3. **Mock 与接口偏离：**
   - "测试通过但运行时因缺少方法崩溃"的情况为零
   - mock 与接口之间没有方法名不匹配

4. **自省：**
   - 可衡量：实现者报告是否包含自省发现？
   - 定性：到达代码审查的 bug 是否更少？

5. **技能阅读：**
   - 子代理报告引用技能的 gate 函数
   - 代码审查中反模式违规更少

---

## 风险与缓解

### 风险：提示词臃肿
**问题：** 添加所有这些要求会让提示词令人应接不暇
**缓解：**
- 分阶段实施（不要一次全部添加）
- 让一些新增内容有条件触发（E2E 卫生仅用于 E2E 测试）
- 为不同任务类型考虑模板

### 风险：分析瘫痪
**问题：** 过多的反思/验证拖慢执行
**缓解：**
- 保持 gate 函数快速（秒级，而非分钟级）
- 初始阶段将精简上下文设为可选
- 监控任务完成时间

### 风险：虚假的安全感
**问题：** 遵循检查清单并不保证正确性
**缓解：**
- 强调 gate 函数是下限，而非上限
- 在技能中保留"运用判断"的表述
- 文档说明技能捕获的是常见失败，而非所有失败

### 风险：技能分歧
**问题：** 不同技能给出相互冲突的建议
**缓解：**
- 审查所有技能的变更以确保一致性
- 文档化技能如何交互（集成章节）
- 在部署前用真实场景测试

---

## 建议

**立即进行阶段 1：**
- verification-before-completion：配置变更验证
- testing-anti-patterns：Mock 与接口偏离
- requesting-code-review：显式文件读取

**在定稿前与 Jesse 测试阶段 2：**
- 获取关于自省影响的反馈
- 验证进程卫生方式
- 确认技能阅读要求值得其开销

**阶段 3 暂缓，待验证后实施：**
- 精简上下文需要真实世界测试
- 实现者修复工作流变更需要仔细评估

这些变更解决了用户记录的真实问题，同时尽量降低使技能变差的风险。
