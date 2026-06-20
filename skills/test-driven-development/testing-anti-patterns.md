# 测试反模式

**何时加载此参考：** 在编写或修改测试、添加 mock，或 tempted 想给生产代码添加仅供测试的方法时。

## 概览

测试必须验证真实行为，而非 mock 行为。mock 是隔离的手段，不是被测的对象。

**核心原则：** 测代码做什么，而非 mock 做什么。

**遵循严格的 TDD 能防止这些反模式。**

## 铁律

```
1. 绝不测 mock 行为
2. 绝不给生产类添加仅供测试的方法
3. 绝不在不理解依赖的情况下 mock
```

## 反模式 1：测试 mock 行为

**违规：**
```typescript
// ❌ 坏：测试 mock 是否存在
test('renders sidebar', () => {
  render(<Page />);
  expect(screen.getByTestId('sidebar-mock')).toBeInTheDocument();
});
```

**为什么错：**
- 你验证的是 mock 能工作，而非组件能工作
- mock 存在时测试通过，不存在时失败
- 对真实行为什么都没告诉你

**你搭档的纠正：** "我们是在测 mock 的行为吗？"

**修正：**
```typescript
// ✅ 好：测真实组件，或者别 mock 它
test('renders sidebar', () => {
  render(<Page />);  // 别 mock sidebar
  expect(screen.getByRole('navigation')).toBeInTheDocument();
});

// 或者如果 sidebar 必须为隔离而 mock：
// 别对 mock 断言——测 Page 在 sidebar 存在时的行为
```

### 关卡函数

```
在对任何 mock 元素断言之前：
  问："我测的是真实组件行为，还是只是 mock 的存在？"

  如果测的是 mock 的存在：
    停下——删掉断言，或取消对该组件的 mock

  改测真实行为
```

## 反模式 2：生产代码中的仅供测试方法

**违规：**
```typescript
// ❌ 坏：destroy() 只在测试里用
class Session {
  async destroy() {  // 看起来像生产 API！
    await this._workspaceManager?.destroyWorkspace(this.id);
    // ... 清理
  }
}

// 在测试里
afterEach(() => session.destroy());
```

**为什么错：**
- 生产类被仅供测试的代码污染
- 若在生产中被意外调用会很危险
- 违反 YAGNI 和关注点分离
- 把对象生命周期和实体生命周期搞混

**修正：**
```typescript
// ✅ 好：由测试工具处理测试清理
// Session 没有 destroy()——它在生产中是无状态的

// 在 test-utils/
export async function cleanupSession(session: Session) {
  const workspace = session.getWorkspaceInfo();
  if (workspace) {
    await workspaceManager.destroyWorkspace(workspace.id);
  }
}

// 在测试里
afterEach(() => cleanupSession(session));
```

### 关卡函数

```
在给生产类添加任何方法之前：
  问："这是否只被测试使用？"

  如果是：
    停下——不要加
    把它放进测试工具里

  问："这个类是否拥有这个资源的生命周期？"

  如果否：
    停下——这个方法放错了类
```

## 反模式 3：不理解就 mock

**违规：**
```typescript
// ❌ 坏：mock 破坏了测试逻辑
test('detects duplicate server', () => {
  // mock 阻止了测试所依赖的 config 写入！
  vi.mock('ToolCatalog', () => ({
    discoverAndCacheTools: vi.fn().mockResolvedValue(undefined)
  }));

  await addServer(config);
  await addServer(config);  // 应该抛错——但不会！
});
```

**为什么错：**
- 被 mock 的方法有测试依赖的副作用（写 config）
- 为了"保险"而过度 mock 破坏了真实行为
- 测试因错误的原因通过，或神秘地失败

**修正：**
```typescript
// ✅ 好：在正确的层级 mock
test('detects duplicate server', () => {
  // mock 慢的部分，保留测试需要的行为
  vi.mock('MCPServerManager'); // 只 mock 慢的服务器启动

  await addServer(config);  // config 被写入
  await addServer(config);  // 检测到重复 ✓
});
```

### 关卡函数

```
在 mock 任何方法之前：
  停下——先别 mock

  1. 问："真实方法有什么副作用？"
  2. 问："这个测试是否依赖其中任何副作用？"
  3. 问："我是否完全理解这个测试需要什么？"

  如果依赖副作用：
    在更低层级 mock（实际的慢/外部操作）
    或使用保留必要行为的测试替身
    而非测试所依赖的高层方法

  如果不确定测试依赖什么：
    先用真实实现跑测试
    观察实际需要发生什么
    然后在正确层级加最小 mock

  红旗：
    - "我为了保险 mock 这个"
    - "这可能慢，最好 mock"
    - 不理解依赖链就 mock
```

## 反模式 4：不完整的 mock

**违规：**
```typescript
// ❌ 坏：部分 mock——只有你以为需要的字段
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' }
  // 缺失：下游代码使用的 metadata
};

// 之后：当代码访问 response.metadata.requestId 时崩溃
```

**为什么错：**
- **部分 mock 隐藏了结构假设** —— 你只 mock 了你知道的字段
- **下游代码可能依赖你没包含的字段** —— 静默失败
- **测试通过但集成失败** —— mock 不完整，真实 API 完整
- **虚假信心** —— 测试对真实行为什么都没证明

**铁律：** mock 与现实中存在的完整数据结构，而非只是你眼前测试用的字段。

**修正：**
```typescript
// ✅ 好：镜像真实 API 的完整性
const mockResponse = {
  status: 'success',
  data: { userId: '123', name: 'Alice' },
  metadata: { requestId: 'req-789', timestamp: 1234567890 }
  // 真实 API 返回的所有字段
};
```

### 关卡函数

```
在创建 mock 响应之前：
  检查："真实 API 响应包含哪些字段？"

  动作：
    1. 从文档/示例检查实际 API 响应
    2. 包含系统下游可能消费的所有字段
    3. 验证 mock 完全匹配真实响应 schema

  关键：
    如果你在创建 mock，你必须理解整个结构
    当代码依赖被省略的字段时，部分 mock 会静默失败

  如果不确定：包含所有有文档的字段
```

## 反模式 5：把集成测试当事后想法

**违规：**
```
✅ 实现完成
❌ 没写测试
"准备好测试了"
```

**为什么错：**
- 测试是实现的一部分，不是可选的后续
- TDD 本会捕获这个
- 没有测试不能声称完成

**修正：**
```
TDD 循环：
1. 写失败测试
2. 实现让它通过
3. 重构
4. 然后才声称完成
```

## 当 mock 变得太复杂时

**警告信号：**
- mock setup 比测试逻辑还长
- 为了让测试通过而 mock 一切
- mock 缺少真实组件才有的方法
- mock 改变时测试就坏

**你搭档的问题：** "我们这里需要用 mock 吗？"

**考虑：** 用真实组件的集成测试往往比复杂 mock 更简单

## TDD 防止这些反模式

**为什么 TDD 有帮助：**
1. **先写测试** → 强迫你想清楚自己到底在测什么
2. **看它失败** → 确认测试测的是真实行为，而非 mock
3. **最小实现** → 不会混入仅供测试的方法
4. **真实依赖** → 在 mock 之前你就看到测试实际需要什么

**如果你在测 mock 行为，你违反了 TDD** —— 你在没有先看测试对真实代码失败的情况下就加了 mock。

## 快速参考

| 反模式 | 修正 |
|--------------|-----|
| 对 mock 元素断言 | 测真实组件，或取消它的 mock |
| 生产中的仅供测试方法 | 移到测试工具 |
| 不理解就 mock | 先理解依赖，最小化 mock |
| 不完整的 mock | 完整镜像真实 API |
| 把测试当事后想法 | TDD——测试优先 |
| 过度复杂的 mock | 考虑集成测试 |

## 红旗

- 断言检查 `*-mock` 测试 ID
- 只在测试文件里被调用的方法
- mock setup 占测试的 >50%
- 移除 mock 时测试就失败
- 说不出为什么需要 mock
- "为了保险"而 mock

## 底线

**mock 是隔离的工具，不是被测的对象。**

如果 TDD 揭示你在测 mock 行为，你就走偏了。

修正：测真实行为，或质疑你到底为什么要 mock。
