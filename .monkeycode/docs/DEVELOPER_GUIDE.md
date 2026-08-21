# DeepSeek Harness 开发者指南

## 项目目的

DeepSeek Harness（`dsh`）是 DeepSeek AI 的开源 Agent 运行时框架，采用"**一切都是插件**"的架构，基于 vendored 的 Cordis 插件框架构建。它在 Agent 领域承担的是"运行时 + 组合层"的角色：不预设固定的 Agent 行为，而是通过插件树把模型适配、工具、持久化、权限、UI 组合成任意形态的可运行 Agent。

**核心职责**：
- 提供可叠加配置层（profile / bundle / patch），把插件组合成可运行的 Agent
- 提供模型主循环（agent-loop）、会话日志、工具管线、提示词装配等产品主干
- 提供能力缝（Service Definition / Provider / Consumer）让每种能力可替换
- 提供三种进程外接口：Web API、TS/Python SDK（JSON-RPC）、ACP

**相关系统**：
- `vendor/`（vendored Cordis/cosmokit/schemastery）— 框架底座，固定版本源码
- `docs/`（架构与子系统文档）— 本仓库自带的权威文档，改动 `packages/` 前必读 `docs/architecture.md`

## 环境搭建

### 前置条件

- Node.js `^22.19 || >=24`（CI 覆盖 22.19 / 24 / 26）
- Corepack 启用的 pnpm（仓库锁定 `pnpm@11.7.0`；`pnpm --version` 不解析时先 `corepack enable`）
- Git 2.26+
- 可选：DeepSeek API key（`DEEPSEEK_API_KEY`），用于 Web/headless/ACP 演示与真实 API e2e

### 安装

```bash
git clone https://github.com/hoobee-1999/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run typecheck
```

`pnpm install` 同时配置 worktree 本地的 Lefthook hooks 与翻译配对 merge driver。若 postinstall 被跳过，手动执行 `node scripts/install-lefthook.mjs`。`pnpm run typecheck` 成功即安装完成。

### 环境变量

| 变量 | 必需 | 描述 | 示例 |
|---|---|---|---|
| `DEEPSEEK_API_KEY` | 真实 API 时必需 | DeepSeek API key | `sk-...` |
| `DEEPSEEK_BASE_URL` | 否 | 覆盖 API 基地址 | `https://...` |
| `DSH_HOME` | 否 | Harness home 目录（默认 `~/.dsh`） | `~/.dsh` |
| `DSH_TELEMETRY_DISABLED` | 否 | 任意非空值禁用遥测 | `1` |

绝不提交真实凭据；`.env` 文件被 gitignore。

### 构建与运行

```bash
pnpm run build       # 完整构建：tsc Host → tsdown Host → tsc Client → tsdown Client → web
pnpm run build:lib   # 只构建 lib（host + client）
pnpm dsh web         # 用已构建产物启动 Web UI（默认 127.0.0.1:3080）
pnpm dsh --profile headless "summarize this workspace"   # 一次性任务（需 DEEPSEEK_API_KEY）
pnpm run demo:cordis  # 自引用 demo：agent 修改自己的插件运行时（需 key）
pnpm run demo:acp     # ACP 自动化服务器（需 key）
```

### 运行测试

```bash
pnpm run test                 # vitest 单元测试
pnpm run test:coverage        # CI 覆盖率门禁：packages/*/*/src 每文件 100%
pnpm run test:e2e             # 真实 API 测试，无 key 自跳过
pnpm run test:snapshot        # keyless ACP/headless 回放快照
pnpm run typecheck            # 类型检查（完整 Host lib phase）
pnpm run lint                 # oxlint
pnpm run hygiene              # knip + publint + workspace constraints + NodeNext 消费检查
pnpm run doc-sync             # 文档门禁合集
pnpm run check:all            # 全部本地门禁
```

## 开发工作流

### 代码质量工具

| 工具 | 命令 | 目的 |
|---|---|---|
| TypeScript | `pnpm run typecheck` | 类型检查（strict） |
| oxlint | `pnpm run lint` | 代码检查 |
| jscpd | `pnpm run duplication` | 跨文件重复检测 |
| vitest | `pnpm run test` | 单元测试 |
| vitest + v8 | `pnpm run test:coverage` | 覆盖率门禁 |
| knip | `pnpm run hygiene` | 未使用依赖检测 |

### 提交前检查（Lefthook 自动）

`pre-commit`：校验 staged 配对记录、用 `.oxlintrc.staged.json` 做 oxlint 修复、必要时重新生成 `THIRD_PARTY_NOTICES.md`、检查空白错误、跑 vendor manifest 守卫。`pre-push`：跑 `pnpm run typecheck`。Hooks 有意不跑完整测试/构建；按改动面挑选相关检查（`dsh-pre-push-checks` skill 指导选择），CI 负责穷尽覆盖。

### 分支策略

- `master` — 主干
- 开发分支：建议遵循仓库的 stack/PR 工作流（多个依赖 PR 用 GitHub 官方 stacked-PR 特性）

### 检查选择原则

| 改动面 | 选择的检查 |
|---|---|
| 行为逻辑 | 聚焦的单元测试 |
| 模型/用户输出 | 快照测试 |
| 文档 | `pnpm run doc-sync` |
| 发布路径 | build + hygiene + 构建后 smokes |
| Provider 行为 | 真实 API e2e |

## 常见任务

### 跑一个 agent 任务（源码启动）

```bash
pnpm run build
pnpm dsh --profile headless "summarize this workspace"
```

### 加一个新工具（模型面能力）

**需修改/创建的文件**：
1. `packages/<group>/tool-<name>/` — 新建工具包，`apply(ctx, config)` 里 `ctx.tools.register(defineTool({...}))`
2. `packages/<group>/tool-<name>/src/index.ts` — 导出 `name` / `inject` / `Config` / `apply`
3. 对应 bundle 的 `cordis.patch.yml` — 用 `insert` 挂载新工具行
4. 测试与快照

**参考**：`docs/cookbook/adding-a-tool.md` 有完整分步指南。工具的 UI 渲染意图（generic/terminal/diff、locations）是设计的一部分，开始就要定。

### 添加一个新的插件包

1. 加入已有 group（`packages/<group>/<pkg>/`），包名 `@deepseek-ai/dsh-<pkg>`
2. 按包规范：`src/types.ts` 只放类型；测试放 `tests/`；extends `tsconfig.base.json`（Client 用 `tsconfig.base.client.json`）
3. 注册到恰一个 aggregate（Host 进 `tsconfig.host.json`，Client 进 `tsconfig.client.json`）
4. 服务包 default-export service 类；函数插件 named-export `name`/`inject`/`Config`/`apply` 且无 default export
5. 写 README（含 Model Experience 段落）与 Known Limitations
6. 每个包提供 `./invariant`（事件/数据关系检查，或给"无运行时不变量"理由）

完整流程见 `docs/cookbook/adding-a-package.md`。

### 实现或替换一个能力缝 provider

以 shell 为例：
1. Service Definition：`packages/shell/shell/src/index.ts`（`abstract class ShellExecutor extends Service`，声明 `ctx.shell`）
2. Provider：新建 `packages/shell/<impl>/src/index.ts`，default-export 继承抽象类的具体类
3. 在 patch 层用一行替换默认 provider 的 `name`
4. 测试：provider 单测 + 消费它的工具集成测试

换一个 provider 就换掉整个产品行为（例如 `bash-sandbox` 把每条命令经 `ctx.sandbox` 围栏）。

### 添加持久会话状态 / 新会话事件

1. 扩展 `SessionEventMap`（`packages/core/session/src/types.ts` 的声明合并映射）
2. 事件的 JSDoc 需要 `@mode` 与载荷 `@param`
3. 从日志渲染与回放：新事件必须可以从日志重建模型可见输入
4. 若改的是公共类型，同步更新 `docs/subsystems/` 页与 TS/Python SDK 预期输出

### 修改 agent-loop

agent-loop 是默认驱动，改动它之前必须先更新 `docs/architecture.md`。新行为优先挂到文档化扩展点（`agent/*`、`tools/*` 事件）而不是改循环本身。

## 编码规范

### 文件组织
- 服务包 default-export service 类；函数插件 named-export 四个字段
- 测试在包级 `tests/`，不在 `src/__tests__/`
- `src/types.ts` 只放类型（无运行时代码）

### 命名

| 类型 | 约定 | 示例 |
|---|---|---|
| 包 | `@deepseek-ai/dsh-<name>` | `@deepseek-ai/dsh-fs` |
| 服务 | 继承 Cordis `Service` | `abstract class FileSystem extends Service` |
| ctx 键 | 小驼峰 | `ctx.tools`、`ctx.shell` |
| 跨边界 id | `Branded<B>`，绝不裸 `string` | `SessionId`、`WorkspaceId` |
| 事件 | `domain/name`（kebab） | `tools/pre-execute`、`session/event` |

### 关键约定
- **注册即副作用**：一切贡献走 `ctx.effect()` / `ctx.on()`；注册返回 disposer
- **waterfall 监听器必须调 `next()`**，否则短路链条
- **模型可见 ⟺ 已记录**：到达模型的任何输入必须能从会话日志重建
- **能力缝必须三角色齐全**：Service Definition / Provider / Consumer
- **不变量断言属有关系**：检查权威事件流或可变数据，而非服务存在性
- **闭环联合用 `assertNever`**；merge 扩展联合落到文档化默认
- **Trust TypeScript at typed same-process boundaries**：不在同进程强类型边界加运行时校验

### 错误处理
- 空 `catch` 必须命名它吞了什么，以及为什么没有别的可达
- 配置错误要 fail loud：能自包含的加载期失败，否则在最早可解析点失败
- 同一异步操作用同一个生命周期控制器/事务表示

### 日志
- 诊断走 stderr（SDK/ACP 的 stdout 只承载协议帧）
- 空 catch 命名吞掉的东西；保留行为、失败、时序、归属事实

## 文档

仓库文档遵循严格的分层（见 `docs/AGENTS.md`）：架构图（`docs/architecture.md`）→ 子系统参考（`docs/subsystems/`）→ Agent Notes（`.agents/notes/`）→ cookbook → package README。每个非平凡改动需要至少一个 Agent Note 在同一 PR 里；`pnpm run doc-sync` 是文档门禁。

本教程文档（`.monkeycode/docs/`）是从代码库梳理出的速览，仓库自带的 `docs/` 是权威文档。
