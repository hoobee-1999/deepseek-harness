# packages/sdk + python

进程外驱动 harness 的协议栈。从另一个进程把运行时作为子进程拉起，通过 stdio 上的换行分隔 JSON-RPC 2.0 驱动 turns。

## 结构

```
packages/sdk/
├── protocol/     # @deepseek-ai/dsh-sdk-protocol — 共享线协议（NDJSON 帧 + 命名类型）
├── client/       # @deepseek-ai/dsh-sdk-client — TypeScript 客户端
└── server/       # @deepseek-ai/dsh-sdk-jsonrpc-server — 运行时侧的 jsonrpc 插件

python/
├── sdk/          # deepseek-harness-sdk（import deepseek_harness）— 高层 API + 低层 client
└── sdk-runtime/  # deepseek-harness-runtime-bin — 打包的 dsh-jsonrpc-agent 可执行文件 + 默认配置
```

## 关键文件

| 文件 | 目的 |
|------|------|
| `packages/sdk/protocol/src/index.ts` | `JsonRpcLineTransport`、`HarnessSdkRequestMap`、`InitializeParams/Result` |
| `packages/sdk/client/src/index.ts` | `DeepSeekHarness`、`HarnessSession`、`RunResult`；低层 `HarnessClient` |
| `packages/sdk/server/src/server.ts` | `HarnessSdkJsonRpcServer` 方法实现（`initialize` / `session/prompt` / `shutdown`） |
| `python/sdk/src/deepseek_harness/api.py` | Python 高层 `DeepSeekHarness` / `Session.run()` |

## 依赖

**本模块依赖**：
- `packages/core/agent` — server 侧注入 `ctx.agents` 驱动 turns
- 运行时产物（`apps/cli` 或 Python bundled runtime）

**依赖本模块的**：
- `packages/subagent/subagent-dsh-sdk` — 用 SDK 做子代理 provider
- 外部集成者：把 harness 嵌入自己的产品

## 规范

### 协议纪律

- stdout 只承载 JSON-RPC 帧，不加载 stdout logger（诊断走 stderr）
- `initialize` 是就绪边界（等 Loader 树安定）
- `shutdown` 刷新响应后 dispose 根 context 并以 0 退出
- **committed output only**：模型原始增量、推理、工具活动不上 wire

### 活动区间

`run(input)` = 入队 → 等 `MessageId` 出现在 durable `agent/inbox/spliced` 回执 → 收集到下一个 whole-agent `idle`。`finish_reason` 是区间内最后 root-session `turn/end` 的 `kind`。

### TS 与 Python 孪生

两者同协议、同分层。差异：Python 默认解析并启动 bundled 运行时；TS 要求显式 `command/args`。`SessionEventMap` 变化时两者预期输出在同 PR 更新。

### 测试

协议单测、client 集成测试与 Python 对应测试在各自 `tests/`。真实 API e2e 需要 `DEEPSEEK_API_KEY`。
