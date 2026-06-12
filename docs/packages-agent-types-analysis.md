# src/types.ts 解析

这个文件定义了整个 agent 包的**核心类型系统**，是理解整个模块的起点。

## 1️⃣ StreamFn（第 24-26 行）— 流函数

Agent 底层用来调用 LLM 的流式函数。关键契约：
- **不能抛异常**，错误必须通过返回的 stream 协议传递
- 可以替换为代理转发（`streamProxy`）

## 2️⃣ ToolExecutionMode（第 36 行）— 工具执行模式

```
"sequential" — 一个一个执行
"parallel"   — 先串行预检，然后并发执行
```

## 3️⃣ QueueMode（第 44 行）— 队列排空模式

```
"all"           — 一次性注入所有排队的消息
"one-at-a-time" — 每次只注入最旧的一条
```
控制 steering（转向）和 follow-up（后续）消息的注入方式。

## 4️⃣ 工具钩子系统（第 55-109 行）— 核心拦截机制

这是最值得关注的设计：

```
beforeToolCall                    afterToolCall
      │                                 │
      ▼                                 ▼
  参数已验证                      工具已执行完毕
  可以阻止执行                      可以修改结果
  返回 { block: true }            返回 AfterToolCallResult
```

| 接口 | 作用 |
|---|---|
| `BeforeToolCallContext` | 传给 `beforeToolCall`：包含 assistant 消息、tool call、已验证的参数、当前上下文 |
| `AfterToolCallContext` | 传给 `afterToolCall`：额外包含工具执行结果和错误标志 |
| `BeforeToolCallResult` | `{ block?, reason? }` — 阻止工具执行 |
| `AfterToolCallResult` | `{ content?, details?, isError?, terminate? }` — 部分覆盖工具结果 |

## 5️⃣ 循环控制钩子（第 112-277 行）— AgentLoopConfig

这是**低层 agent loop 的配置对象**，包含所有可注入的扩展点：

```
AgentLoopConfig
├── model                          — 使用哪个模型
├── convertToLlm                   — AgentMessage → LLM Message 转换器
├── transformContext               — 消息预处理（裁剪、注入外部上下文）
├── getApiKey                      — 动态 API Key（如 OAuth 刷新）
├── shouldStopAfterTurn            — 是否在完成一轮后停止（用于压缩前判断）
├── prepareNextTurn                — 准备下一轮的上下文/模型/思考级别
├── getSteeringMessages            — 获取转向消息（运行时干预）
├── getFollowUpMessages            — 获取后续消息（任务队列）
├── toolExecution                  — 执行模式（parallel/sequential）
├── beforeToolCall                 — 工具执行前拦截
└── afterToolCall                  — 工具执行后拦截
```

**关键契约**：所有钩子函数**不能抛异常**，必须返回安全的 fallback 值。

## 6️⃣ ThinkingLevel（第 284 行）— 思考级别

```
"off" | "minimal" | "low" | "medium" | "high" | "xhigh"
```
用于支持推理模型（如 Claude 的 extended thinking）。

## 7️⃣ AgentMessage（第 309 行）— 消息联合类型

```typescript
AgentMessage = LLM标准消息 | 自定义消息
```

通过 TypeScript **声明合并**扩展自定义消息类型：
```typescript
declare module "@earendil-works/pi-agent-core" {
  interface CustomAgentMessages {
    notification: { role: "notification"; text: string };
  }
}
```

## 8️⃣ AgentState（第 317-342 行）— 公共状态接口

Agent 的公开状态：
| 属性 | 说明 |
|---|---|
| `systemPrompt` | 系统提示词 |
| `model` | 当前模型 |
| `thinkingLevel` | 思考级别 |
| `tools` | 可用工具列表（setter 会拷贝数组） |
| `messages` | 对话记录（setter 会拷贝数组） |
| `isStreaming` | 是否正在处理中 |
| `streamingMessage` | 当前流式消息 |
| `pendingToolCalls` | 正在执行的 tool call IDs |
| `errorMessage` | 最近一次的错误信息 |

## 9️⃣ AgentTool（第 361-384 行）— 工具定义

```typescript
interface AgentTool {
  name: string;           // 工具名
  label: string;          // UI 显示名
  description: string;    // 描述
  parameters: TSchema;    // TypeBox 参数 schema
  prepareArguments?: ...; // 参数适配（兼容旧格式）
  execute: (...);         // 执行函数
  executionMode?: ...;    // 单独覆盖执行模式
}
```

## 🔟 AgentEvent（第 403-418 行）— 事件联合类型

完整的事件流：
```
agent_start
  → turn_start
    → message_start (user)
    → message_end (user)
    → message_start (assistant)
    → message_update × N  (流式更新)
    → message_end (assistant)
    → tool_execution_start
    → tool_execution_update × N
    → tool_execution_end
  → turn_end
agent_end
```

## 与 agent.ts 的对应关系

| types.ts 定义 | agent.ts 使用位置 |
|---|---|
| `AgentState` | `_state` 私有字段 + `state` getter |
| `AgentEvent` | `processEvents(event)` 的 switch 处理 |
| `AgentLoopConfig` | `createLoopConfig()` 构造 |
| `AgentContext` | `createContextSnapshot()` 构造 |
| `ToolExecutionMode` | `toolExecution` 属性 |
| `BeforeToolCallResult` / `AfterToolCallResult` | 直接传递给 agent-loop |
| `QueueMode` | `steeringQueue` / `followUpQueue` 的行为 |

---

## 💡 总结

这个文件是整个 agent 包的**类型骨架**。理解了它之后，再看 `agent.ts` 和 `agent-loop.ts` 的实现会非常清晰。
