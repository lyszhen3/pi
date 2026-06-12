# packages/agent 学习指南

## 模块结构总览

```
packages/agent/
├── src/
│   ├── agent.ts           ← Agent 类主入口
│   ├── agent-loop.ts      ← 底层循环逻辑（prompt → LLM → tools）
│   ├── proxy.ts           ← 代理/后端转发
│   ├── types.ts           ← 核心类型定义
│   ├── index.ts           ← 公共导出
│   └── harness/           ← Harness 运行环境
│       ├── agent-harness.ts      ← 完整的 Agent 运行框架
│       ├── system-prompt.ts      ← 系统提示词构建
│       ├── prompt-templates.ts   ← 提示词模板
│       ├── skills.ts             ← 技能系统
│       ├── messages.ts           ← 消息格式
│       ├── types.ts              ← Harness 类型
│       ├── session/              ← 会话管理（持久化）
│       │   ├── session.ts        ← 会话核心
│       │   ├── jsonl-storage.ts  ← JSONL 存储
│       │   ├── memory-storage.ts ← 内存存储
│       │   ├── jsonl-repo.ts     ← 代码仓库快照
│       │   └── memory-repo.ts    ← 内存仓库
│       ├── compaction/           ← 对话压缩（上下文管理）
│       │   ├── compaction.ts     ← 压缩逻辑
│       │   └── branch-summarization.ts ← 分支摘要
│       └── utils/
│           ├── truncate.ts       ← 文本截断
│           └── shell-output.ts   ← Shell 输出处理
├── docs/                  ← 详细文档
│   ├── agent-harness.md
│   ├── durable-harness.md
│   ├── hooks.md
│   └── observability.md
└── test/                  ← 测试（配合代码看测试最直观）
```

## 推荐学习顺序

### Level 1: 核心概念（1-2 小时）

1. **`src/types.ts`** — 理解 `AgentMessage`、`AgentTool`、事件类型
2. **`src/agent.ts`** — Agent 类的实现，状态管理、事件订阅
3. **`src/agent-loop.ts`** — 底层循环：用户输入 → LLM 调用 → 工具执行 → 回复

### Level 2: Harness 运行环境（2-3 小时）

4. **`src/harness/types.ts`** — Harness 类型定义
5. **`src/harness/system-prompt.ts`** — 系统提示词如何构建
6. **`src/harness/agent-harness.ts`** — 完整的 Agent 运行框架
7. **`src/harness/session/session.ts`** — 会话生命周期管理

### Level 3: 高级特性（按需深入）

8. **`src/harness/compaction/`** — 对话压缩（当上下文太长时怎么办）
9. **`src/harness/skills.ts`** — 技能系统（如何让 Agent 学会特定任务）
10. **`src/proxy.ts`** — 代理模式（浏览器 ↔ 后端转发）

### Level 4: 文档 + 测试

11. 读 `docs/` 下的 4 篇文档
12. 配合 `test/` 下的测试用例理解具体行为

---

## 最快上手方法

**建议先看测试文件**，测试是最直接的"使用说明书"：
- `test/agent.test.ts` — Agent 基本用法
- `test/agent-loop.test.ts` — 循环逻辑
- `test/harness/agent-harness.test.ts` — 完整运行框架
