# src/agent.ts 解析

`agent.ts` 把 `types.ts` 中定义的所有类型**组装成一个可运行的 Agent 类**。它是状态管理、事件发布、队列管理和生命周期控制的集合体。

## 整体架构

```
Agent 类
│
├── 内部状态 (_state)         — 可变状态的封装
├── 事件监听 (listeners)       — 订阅/发布机制
├── 两个队列                   — steeringQueue + followUpQueue
├── 运行控制 (activeRun)       — 管理当前正在执行的 run
│
└── 公共属性                   — 直接映射到 AgentLoopConfig
    ├── convertToLlm
    ├── beforeToolCall
    ├── afterToolCall
    ├── streamFn
    └── ...
```

## 1️⃣ 构造函数 + 状态初始化（第 66-93, 201-219 行）

### `createMutableAgentState` — 创建"带保护"的状态对象

关键点在 **getter/setter**：

```typescript
get tools() { return tools; }
set tools(nextTools) { tools = nextTools.slice(); }  // ← 赋值时自动拷贝
```

这是防御性编程——防止外部代码不小心篡改内部状态。

### 构造函数 — 接收 `AgentOptions`，设置默认值

- `convertToLlm` → 默认过滤出 LLM 能理解的消息（user/assistant/toolResult）
- `streamFn` → 默认用 `streamSimple`（直连 LLM）
- `steeringMode` / `followUpMode` → 默认 `one-at-a-time`
- `toolExecution` → 默认 `parallel`

## 2️⃣ PendingMessageQueue（第 118-152 行）— 内部队列类

一个简单的消息队列，支持两种排空策略：

```
"all"           → drain() 返回所有消息，清空队列
"one-at-a-time" → drain() 只返回最旧的一条
```

有两个实例：
- **`steeringQueue`** — 运行时转向（agent 正在工作时插入新指令）
- **`followUpQueue`** — 后续任务（agent 停下来后才注入）

### steering vs followUp 的区别

```
Agent 正在工作（跑 tool calls）
  │
  ├── steer(msg) → 注入 steeringQueue
  │   → 当前 turn 的 tool calls 执行完后
  │   → 注入 steering 消息，继续下一个 turn
  │
  └── followUp(msg) → 注入 followUpQueue
      → agent 完全停下来后（没有更多 tool calls 和 steering）
      → 才注入 followUp 消息
```

## 3️⃣ 核心方法 — 用户入口

| 方法 | 行号 | 作用 |
|---|---|---|
| `subscribe(fn)` | 231 | 订阅事件，返回取消函数 |
| `prompt(input)` | 325-335 | 发起新对话（支持字符串/消息/消息数组） |
| `continue()` | 338-365 | 从当前上下文继续 |
| `steer(msg)` | 264 | 入队转向消息 |
| `followUp(msg)` | 269 | 入队后续消息 |
| `abort()` | 300 | 中断当前运行 |
| `waitForIdle()` | 309 | 等待当前 run 完全结束 |
| `reset()` | 314 | 清空一切 |

## 4️⃣ `prompt()` 的完整链路

```
prompt("你好")
  ↓
normalizePromptInput("你好")     → [{ role: "user", content: "你好", timestamp: ... }]
  ↓
runPromptMessages(messages)
  ↓
runWithLifecycle(async (signal) => {
    runAgentLoop(                ← 调用底层生成器
      messages,
      createContextSnapshot(),   ← 当前状态快照
      createLoopConfig(),        ← 配置对象
      (event) => processEvents,  ← 事件回调
      signal,
      streamFn
    )
  })
```

## 5️⃣ `runWithLifecycle` — 生命周期包装器（第 451-474 行）

**这是所有 prompt/continue 的公共入口**：

```
1. 创建 AbortController + Promise（存为 activeRun）
2. 设置 isStreaming = true
3. 执行 executor
   ├── 成功 → 正常退出
   └── 失败 → handleRunFailure() 发送错误事件流
4. finally → finishRun() 清理
```

### `handleRunFailure` — 错误处理（第 476-492 行）

当 agent loop 抛出异常时，不会直接崩溃：
1. 造一个"假的" assistant 消息（`stopReason: "error"` 或 `"aborted"`）
2. 发出完整的 `message_start → message_end → turn_end → agent_end` 事件流
3. UI 能正常显示错误，不会卡住

## 6️⃣ `processEvents` — 事件处理中枢（第 509-556 行）

每个从底层 loop 发出的事件都走这里：

```
event 进来
  ↓
switch(event.type) {
  message_start   → 保存 streamingMessage
  message_update  → 更新 streamingMessage
  message_end     → 清除 streamingMessage, 推入 messages[]
  tool_start      → pendingToolCalls.add(id)
  tool_end        → pendingToolCalls.delete(id)
  turn_end        → 记录 errorMessage
  agent_end       → 清除 streamingMessage
}
  ↓
for (listener of listeners) await listener(event, signal)
```

**关键设计**：
- 先更新内部状态
- 再**同步等待**所有 listener 完成（barrier 语义）
- `agent_end` 的 listener 完成后，`finishRun()` 才清理 `activeRun`

## 7️⃣ `continue()` 的特殊逻辑（第 338-365 行）

```
最后一轮消息是什么？
  ├── assistant  → 先检查 steeringQueue，有就注入
  │              → 再检查 followUpQueue，有就注入
  │              → 都没有 → 报错（不能在 assistant 后继续）
  └── user/toolResult → runContinuation() 继续
```

## 8️⃣ `createLoopConfig` — 配置转换（第 422-449 行）

把 Agent 类的属性转换为 `AgentLoopConfig` 传给低层 loop：

```typescript
createLoopConfig() {
  return {
    model: this._state.model,
    convertToLlm: this.convertToLlm,
    beforeToolCall: this.beforeToolCall,
    afterToolCall: this.afterToolCall,
    getSteeringMessages: () => this.steeringQueue.drain(),
    getFollowUpMessages: () => this.followUpQueue.drain(),
    // ...
  }
}
```

## 一句话总结

> **Agent 类不跑 LLM 也不执行工具** —— 它管的是**状态、事件、队列、生命周期**。真正的活都委托给了底层的 `runAgentLoop`。
