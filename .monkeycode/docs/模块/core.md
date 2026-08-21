# packages/core

产品 API 主干。这些包构成运行时的心智模型：会话、工具、提示词、Agent 与主循环。

## 结构

```
packages/core/
├── session/          # @deepseek-ai/dsh-session — 追加式 SessionEvent 日志与 SessionStore
├── tools/            # @deepseek-ai/dsh-tools — 作用域化工具注册表 + 受守卫执行管线
├── agent/            # @deepseek-ai/dsh-agent — Agent 接口、活体注册表、agent/* 事件
├── agent-loop/       # @deepseek-ai/dsh-agent-loop — 默认循环实现
├── agent-default-model/  # 默认模型选择
├── agent-tool-presentation/  # 工具结果物化（deep-freeze JSON）
├── system-prompt/    # @deepseek-ai/dsh-system-prompt — 提示词 section 与 schema 装配
└── scope/            # @deepseek-ai/dsh-scope — 每-agent 作用域注册原语（纯库）
```

## 关键文件

| 包 | 文件 | 目的 |
|------|------|------|
| session | `packages/core/session/src/types.ts` | `SessionId`、`SessionEventMap`、`SessionEvent`（判别联合） |
| session | `packages/core/session/src/index.ts` | `Session`（内存日志）+ `SessionStore`（`ctx.sessions`） |
| tools | `packages/core/tools/src/schema.ts` | `defineTool` |
| tools | `packages/core/tools/src/types.ts` | `ToolDefinition`、`ToolExecution`、`ToolExecutionResult`、`PreToolDecision` |
| tools | `packages/core/tools/src/index.ts` | `ToolRuntime`（`ctx.tools`）+ `tools/*` 管线事件 |
| agent | `packages/core/agent/src/index.ts` | `AgentRegistry`（`ctx.agents`）+ `agent/created` / `agent/disposed` |
| agent-loop | `packages/core/agent-loop/src/index.ts` | `AgentLoop extends Service implements AgentFactory`（`ctx.agentLoop`） |
| system-prompt | `packages/core/system-prompt/src/index.ts` | `SystemPrompt`（`ctx.systemPrompt`）：`section()` / `variable()` / `assemble()` |
| scope | `packages/core/scope/src/` | 每-agent 作用域注册、scoped dispatch、shadowing |

## 依赖

**本模块依赖**：
- `@deepseek-ai/cordis`（peer）— 框架底座
- `packages/util/*` — `Branded<B>` 等零依赖工具

**依赖本模块的**：
- 几乎所有产品包：能力缝 consumer、`packages/session/*`、`packages/host/`、`packages/bundle/*`

## 规范

### 工具注册

```ts
ctx.tools.register(defineTool({
  name: 'my_tool',
  schema: { /* zod schema */ },
  async execute(args, ctx) { /* ... */ },
}))
```

### 工具管线事件

`tools/pre-execute`（waterfall，策略闸门，返回 `PreToolDecision`）→ `tools/execute`（waterfall，实际派发）→ `tools/post-execute`（waterfall，结果后处理）→ `tools/result`（emit，通知）。waterfall 监听器必须调 `next()`。

### 事件声明

新会话事件扩展 `SessionEventMap`；新 agent 事件带 `@mode` JSDoc 与载荷 `@param`。

### 测试

每个包的 `tests/` 覆盖事件与数据关系。相关不变量注册在各包 `./invariant`。模型可见行为需要快照或 e2e。
