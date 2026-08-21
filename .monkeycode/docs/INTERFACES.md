# DeepSeek Harness 接口文档

DeepSeek Harness 是一个平台型仓库，接口横跨多个层面：CLI 命令、Web API 网关、SDK 协议、ACP 协议、插件扩展点，以及 cordis.yml 配置接口。按用途索引如下。

## 1. CLI 命令（`dsh`）

`dsh` bin 由 `apps/cli` 提供。源码启动 `pnpm dsh`，npm 安装后 `npx @deepseek-ai/dsh`。

| 命令 | 说明 |
|---|---|
| `dsh web [args...]` | 启动 Web profile（`--profile web` 的别名），默认 `http://127.0.0.1:3080` |
| `dsh --profile <name> [args...]` | 启动命名 profile（`web`、`headless`，或用户自建） |
| `dsh --profile headless "task"` | 一次性任务：跑完任务、输出最终回复后以 0/1 退出 |
| `dsh plugin --profile <name> <pnpm args...>` | 在 profile 目录转发 pnpm（add/remove/why），按已安装状态对账 bundles |
| `dsh --profile web --dump-config` | 打印组合后的配置树后退出（含用户层与 `--patch`） |
| `dsh --profile web --dump-default-config` | 只打印 bundle 层的配置树 |

通用 flag：`--patch <path>`（可重复，追加 patch overlay）；`--no-open`（web 下不自动开浏览器）；`--host`、`--port`、`--trusted-host`（web 启动参数，透传给 app）。

关键设计：launcher 只解析自己开头的 flags，第一个不识别的 token 之后全部透传给被启动的 app。

## 2. Web API（浏览器 ↔ host）

Web GUI 的传输契约由 `packages/host/apiproxy`（wire 契约）+ `packages/client/connection`（浏览器 carrier）定义。默认 `127.0.0.1:3080`。

| 通道 | 方法 | 说明 |
|---|---|---|
| HTTP POST `/api/<method>` | unary request | body 为 `ClientRequest` 信封，响应 `ServerResponse` |
| HTTP POST `/api/respond` | 应答 server 发起的请求 | body 为 `ClientResponse` |
| WebSocket `/api/events.mux` | 下行事件流 | session 事件 mux 帧：`session/queue`、`session/jobs`、`session/projection` 等 |
| WebSocket `/api/events.host` | 下行 host 事件 | `host/session-added`、`host/workspace-changed` 等 |

方法按域注册在 `RpcMethodMap`：`SessionsApi`（如 `session.prompt`）、`HostApi`（如 `host.describe`）、`EventsApi`。响应总是回显请求的 `rpcId`。

安全围栏：每请求校验 `Host` 头为 loopback 或 `trustedHosts` 条目（DNS-rebinding 防御）；`settings.*`、`credentials.*`、`agentPreset.*` 等特权方法集走 loopback-pinned 通道。

## 3. SDK 协议（TS / Python）

进程外驱动 harness 的私有协议：换行分隔的 JSON-RPC 2.0，走子进程 stdio。协议定义在 `packages/sdk/protocol`，TS 客户端在 `packages/sdk/client`，server 插件在 `packages/sdk/server`；Python 镜像同一 wire 形状（`python/sdk`）。

**client → server**

| 方法 | 参数 | 返回 |
|---|---|---|
| `initialize` | `{ maxTokens? }` | `{ serverInfo: { name, version }, ... }`；`name` 固定为 wire 稳定的 `deepseek-harness-sdk-runtime` |
| `session/prompt` | `{ sessionId?, input, ... }` | 入队回执 `{ messageId }` |
| `shutdown` | — | 刷新响应后 dispose 根 context 并以 0 退出 |

**server → client（通知）**

| 通知 | 载荷 |
|---|---|
| `session.event` | 每个 session 的完整 session-log envelope（不过滤） |
| `session.status` | whole-agent `running` / `idle` 切换 |
| `subagent.started` / `subagent.finished` | 子代理谱系通知 |

### TypeScript 客户端

```ts
import { DeepSeekHarness } from '@deepseek-ai/dsh-sdk-client'

const harness = new DeepSeekHarness({
  launch: { command: 'node', args: ['lib/bin.js', 'cordis.yml'] },
  provider: 'deepseek-official',
  model: 'deepseek-v4-flash',
  maxTokens: 49152,
})

const result = await harness.run('write a hello world', {
  onNotification: (n) => console.log(n),
})
console.log(result.finalResponse)
await harness.close()
```

`run(input, opts)` 的活动区间：入队 → 等 `MessageId` 出现在 durable `agent/inbox/spliced` 回执 → 收集到下一个 whole-agent `idle`。`close()` 走 `shutdown` → stdin-EOF → SIGTERM → SIGKILL 梯子。

### Python 客户端

```python
from deepseek_harness import DeepSeekHarness

with DeepSeekHarness(provider="deepseek-official", model="deepseek-v4-flash",
                     max_tokens=49152) as harness:
    result = harness.run("write a hello world")
    print(result.final_response)
```

Python 默认解析并启动 bundled 运行时（安装 `deepseek-harness-sdk` 会带上 `deepseek-harness-runtime-bin` 平台 wheel）；TS 侧要求显式 `command/args`。

## 4. ACP 协议

Agent Client Protocol 自动化服务器（`packages/acp/acp`），基于 `@agentclientprotocol/sdk`。程序化客户端可创建全新 agent 会话、收发文本/图像、解析一次性权限请求、取消工作。

| 方法 | 说明 |
|---|---|
| `initialize` | 版本协商，按需广告图像能力 |
| `authenticate` | no-op |
| `session/new` | 绝对 cwd；拒绝 additionalDirectories / MCP servers |
| `session/prompt` | 保序文本/图像，等待 admission + whole-agent idle + 有序输出投递 |
| `session/cancel` | 取消会话工作 |
| `session/update` | 每 committed block 发一个 `agent_message_chunk` |
| `session/request_permission` | `allow-once` / `reject-once` 一次性选择 |

生命周期：客户端断开 = 释放该连接的全部 session；每个 session 有独立 prompt 槽、cwd、取消路径与 disposer。仓库内主要客户端是 `dsh-subagent-acp`（子代理 provider）。

## 5. 插件扩展点

插件是扩展 dsh 的主要接口。贡献通过 `ctx.effect()` / `ctx.on()` / `ctx.waterfall()` 完成；注册返回 disposer。

### 5.1 服务（`ctx.<key>`）

| 服务 | 键 | 用途 |
|---|---|---|
| `SessionStore` | `ctx.sessions` | 会话生命周期与追加式日志 |
| `SystemPrompt` | `ctx.systemPrompt` | `section()` 注册有序段、`variable(name, provider)`、`assemble(context)` |
| `ToolRuntime` | `ctx.tools` | `register(defineTool({...}))` 注册工具、`restrict()`/`guard()` 限权 |
| `AgentRegistry` | `ctx.agents` | Agent 注册、inbox、事件 |
| `AgentLoop` | `ctx.agentLoop` | 默认循环实现 |
| LLM 服务 | `ctx.llm` | `LlmAdapter` 注册、`llm/stream` waterfall |
| 能力缝服务 | `ctx.fs` `ctx.shell` `ctx.web` `ctx.subprocess` `ctx.sandbox` 等 | 各能力抽象服务 |

### 5.2 事件

| 事件域 | 模式 | 用途 |
|---|---|---|
| `session/*` | emit/parallel | 持久会话事实：`session/created`、`session/event`、`session/flush` |
| `agent/*` | 混合 | 活体 agent 生命周期：`agent/pre-step`(waterfall)、`agent/request`(waterfall)、`agent/turn-stopping`(serial) |
| `tools/*` | waterfall/emit | 工具管线：`tools/pre-execute`、`tools/execute`、`tools/post-execute`、`tools/result` |
| `llm/stream` | waterfall | 模型流 |
| `fs/*` | waterfall/emit | 文件策略：`fs/write-intent`、`fs/edit-intent`、`fs/observed` |
| `telemetry/*` | emit | 遥测 |

**waterfall 监听器必须调用 `next()`**，否则短路整条链。

### 5.3 工具管线决策

`tools/pre-execute` 返回 `PreToolDecision`：`{kind:'allow'}`、`{kind:'deny', reason}`、`{kind:'ask', reason?}`（approval/权限在此挂钩）。`tools/post-execute` 返回 `PostToolDecision`：`{kind:'accept', content/value}` 或 `{kind:'block', feedback}`。

## 6. 配置接口（cordis.yml）

叶子配置（`examples/*/cordis.yml`）是顶层 entry 数组：

```yaml
- id: llm-deepseek
  name: '@deepseek-ai/dsh-llm-deepseek'
  config:
    thinking: enabled
- id: bash
  name: '@deepseek-ai/dsh-bash-local'
  config: { timeoutMs: 60000 }
```

patch 文件（bundle / profile / `--patch` overlay）是 PatchOptions 数组：

| 形式 | 语义 |
|---|---|
| `{ id: X, config: {...} }` | 整行替换目标行 config（不合并，需写全所有键） |
| `{ id: X, disabled: true }` | 禁用某行 |
| `{ insert: [ {...}, ... ] }` | 追加新行 |

字段：`id`（跨层寻址的行标识）、`name`（插件包名；`cordis:` 前缀为 builtin）、`config`（支持 `!!js` 表达式：`process.env.X`、`dshHomePath('...')`、`ctx.<service>.<field>`）、`inject`（需要的服务注入列表）、`disabled`。

语义要点：patch 对 config 是替换而非合并；同一行的最后一个写者获胜；缺失的 patch 文件静默跳过，但存在却无法解析的 patch 文件必须 fail loud。
