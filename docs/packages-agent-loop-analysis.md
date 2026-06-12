# src/agent-loop.ts 解析

这个文件是整个 agent 包的**核心引擎**——真正跑 LLM 调用和工具执行的地方。

## 一、文件定位

```
agent.ts (状态管理、事件发布)
    ↓ 调用
agent-loop.ts (真正的循环：调 LLM → 执行工具 → 再调 LLM)
    ↓ 调用
@earendil-works/pi-ai (底层 LLM API)
```

## 二、两组入口函数

文件提供了**两套 API**，对应不同的使用场景：

| API | 返回类型 | 使用场景 |
|---|---|---|
| `agentLoop()` / `agentLoopContinue()` | `EventStream` | 流式消费事件（`for await`） |
| `runAgentLoop()` / `runAgentLoopContinue()` | `Promise<AgentMessage[]>` | 一次性拿结果（agent.ts 用这个） |

```
agentLoop()                    → EventStream（底层生成器 API）
  └── 内部调用 runAgentLoop()  → Promise（高级 API）
```

## 三、`runAgentLoop` vs `runAgentLoopContinue`

| | `runAgentLoop` | `runAgentLoopContinue` |
|---|---|---|
| 用途 | 新对话 | 从现有上下文继续（重试） |
| 新增消息 | 把 prompts 加入 context | 不加新消息 |
| 前置检查 | 无 | 最后一条必须是 user/toolResult |
| 事件 | 先发 prompt 的 message_start/end | 不发 prompt 事件 |

两者最终都调用同一个核心函数：**`runLoop`**。

## 四、`runLoop` — 核心主循环（第 155-269 行）

这是整个文件最重要的部分。**双层循环**结构：

```
while (true) {                          ← 外层：处理 follow-up 消息
    while (hasMoreToolCalls || pending) { ← 内层：处理工具调用 + steering
        1. 注入 pending 消息
        2. 调用 LLM（streamAssistantResponse）
        3. 检查错误/中断 → 退出
        4. 执行工具调用（executeToolCalls）
        5. 发 turn_end 事件
        6. prepareNextTurn 钩子
        7. shouldStopAfterTurn 检查 → 退出
        8. 获取 steering 消息
    }
    
    检查 follow-up 消息 → 有就继续外层循环
    没有 → break
}
发 agent_end 事件
```

### 完整执行流程

```
agent_start
  ┌─ turn_start ──────────────────────────────┐
  │  注入 pending 消息 (steering/followUp)    │
  │  message_start/end × N (pending 消息)     │
  │  streamAssistantResponse (调 LLM)         │
  │    message_start                          │
  │    message_update × N (流式)              │
  │    message_end                            │
  │  检查 stopReason                          │
  │    ├── error/aborted → turn_end → 退出    │
  │    └── 正常 → 检查 tool calls             │
  │  executeToolCalls                         │
  │    tool_execution_start                   │
  │    tool_execution_update × N              │
  │    tool_execution_end                     │
  │    message_start/end (tool result)        │
  │  turn_end                                 │
  │  prepareNextTurn 钩子                     │
  │  shouldStopAfterTurn 检查                 │
  │  获取 steering 消息                        │
  └───────────────────────────────────────────┘
  检查 follow-up 消息
  ├── 有 → pendingMessages = followUp, 继续外层
  └── 没有 → break
agent_end
```

## 五、`streamAssistantResponse` — 调用 LLM（第 275-368 行）

这个函数负责**把 AgentMessage 转成 LLM 能懂的 Message，然后调 API**：

```
1. transformContext(messages)        ← 可选：裁剪、注入上下文
2. convertToLlm(messages)            ← 必须：AgentMessage → Message
3. 构建 llmContext (systemPrompt + messages + tools)
4. getApiKey(provider)               ← 动态获取 API Key
5. streamFunction(model, context)    ← 调 LLM
6. 遍历流式事件，发 message_start/update/end
7. 返回最终的 AssistantMessage
```

关键细节：
- `context.messages` 里会**实时替换** partial message（第 333 行），保证上下文始终是最新的
- 如果流没有 start 事件就结束了（异常），会补发 `message_start`

## 六、工具执行 — 两种模式（第 373-516 行）

```
executeToolCalls()
  ├── 任一工具有 executionMode: "sequential" → 顺序执行
  ├── config.toolExecution === "sequential"  → 顺序执行
  └── 否则                                    → 并行执行
```

### 顺序执行 (`executeToolCallsSequential`)

```
for each toolCall:
  emit tool_execution_start
  prepare → execute → finalize
  emit tool_execution_end
  emit tool_result message
  如果 signal.aborted → break
```

每个工具**依次**完成，前一个结束才开始下一个。

### 并行执行 (`executeToolCallsParallel`)

```
for each toolCall:
  emit tool_execution_start
  prepare (串行预检)
    ├── immediate → 直接 emit end，跳过执行
    └── prepared → 推入执行队列（延迟执行）

Promise.all(队列)                    ← 并发执行
  emit tool_execution_end (完成顺序)

按原始顺序 emit tool_result messages ← 保持 assistant 消息中的顺序
```

关键设计：
- **预检是串行的**（`beforeToolCall` 按顺序执行）
- **实际执行是并发的**（`Promise.all`）
- **`tool_execution_end` 按完成顺序发出**
- **`toolResult` 消息按 assistant 原始顺序发出**

## 七、工具调用的 3 阶段生命周期

每个工具调用经过 **prepare → execute → finalize** 三个阶段：

```
prepareToolCall()                    ← 第 1 阶段：预检
  ├── 工具不存在 → immediate error
  ├── prepareArguments (参数适配)
  ├── validateToolArguments (schema 验证)
  ├── beforeToolCall 钩子
  │     └── { block: true } → immediate error
  ├── signal.aborted → immediate error
  └── 返回 { kind: "prepared" }

executePreparedToolCall()            ← 第 2 阶段：执行
  ├── tool.execute(id, args, signal, onUpdate)
  ├── 收集 update 事件
  ├── 成功 → { result, isError: false }
  └── 抛异常 → { errorResult, isError: true }

finalizeExecutedToolCall()           ← 第 3 阶段：后处理
  ├── afterToolCall 钩子
  │     └── 可覆盖 content / details / isError / terminate
  └── 返回 FinalizedToolCallOutcome
```

## 八、终止机制

有 3 种方式让循环提前退出：

| 机制 | 触发条件 | 行为 |
|---|---|---|
| `stopReason` | LLM 返回 error/aborted | 发 turn_end → agent_end → 退出 |
| `shouldStopAfterTurn` | 钩子返回 true | 发 agent_end → 退出 |
| `terminate: true` | 工具结果标记终止 | 当批次**全部**标记才生效 |

`shouldTerminateToolBatch`：
```typescript
// 只有批次里每个工具都返回 terminate: true 才终止
finalizedCalls.length > 0 && finalizedCalls.every(f => f.result.terminate === true)
```

## 九、辅助函数速查

| 函数 | 行号 | 作用 |
|---|---|---|
| `createAgentStream` | 145 | 创建 EventStream 适配器 |
| `prepareToolCallArguments` | 548 | 适配旧格式参数 |
| `createErrorToolResult` | 710 | 构造错误结果 |
| `emitToolExecutionEnd` | 717 | 发 tool_execution_end 事件 |
| `createToolResultMessage` | 727 | 构造 toolResult 消息 |
| `emitToolResultMessage` | 739 | 发 toolResult 的 start/end 事件 |

## 十、与 agent.ts 的关系

```
agent.ts                          agent-loop.ts
────────                          ─────────────
Agent.prompt()                    
  → runPromptMessages()           
    → runWithLifecycle()          
      → runAgentLoop()  ────────→ runAgentLoop()
                                    → runLoop()
                                      → streamAssistantResponse()
                                      → executeToolCalls()
                                        → prepareToolCall()
                                        → executePreparedToolCall()
                                        → finalizeExecutedToolCall()
      ← emit(event)  ──────────── emit 回调
      → processEvents()           
        → 更新状态 + 通知 listeners
```
