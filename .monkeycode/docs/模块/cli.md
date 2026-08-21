# apps/cli

`dsh` 命令行入口。它把 profile 启动、插件管理与 Web UI 别名合成为一个 bin。

## 结构

```
apps/cli/
├── src/bin.ts              # 入口：解析参数 → 加载分层环境 → 按模式动态 import
├── src/args.ts             # Commander 适配器，产出 DshInvocation（判别联合）
├── src/profile-boot.ts     # profile 装配：prepareProfile / composeProfile / runProfile
├── src/plugin.ts           # dsh plugin 的 pnpm 转发器 + bundles 对账
├── src/dump-config.ts      # --dump-config 渲染（逐层 provenance）
├── src/process-shutdown.ts # 有界关闭控制器（SIGTERM→0 / SIGINT→130）
└── src/index.ts            # 导出 bin 与 bootstrap
```

## 关键文件

| 文件 | 目的 |
|------|------|
| `bin.ts` | 53 行的自执行入口；按 `invocation.mode` 动态 import `profile-boot` / `plugin` / `dump-config` |
| `args.ts` | `ProfileInvocation \| DumpConfigInvocation \| PluginInvocation` 三个判别联合 |
| `profile-boot.ts` | 计算完整 patch 栈（bundle → profile → home → overlay），把空根配置重写为 `[]`，watch 用户 patch 热重载 |
| `plugin.ts` | 跑完 pnpm 后按已安装状态对账 `dsh.profile.bundles` |

## 依赖

**本模块依赖**：
- `packages/boot/app-boot` — `boot()` 启动序列、分层环境、patch 解析
- `packages/boot/cmdline` — `provideCmdline` / `parseCmdline`
- `@deepseek-ai/cordis-plugin-loader` 等 vendored 插件

**依赖本模块的**：
- 发布产物 `dsh` bin（npm）与 `npx @deepseek-ai/dsh web` 启动入口
- `packages/bundle/web-app` 的 `webStartup`（经 `ctx.cmdlineArgs` 消费 CLI flags）

## 规范

### 关键命令

- `dsh web` = `--profile web` 别名；`dsh --profile <name>` 启动任意 profile
- `dsh plugin --profile <name> <pnpm args...>` 管理 out-of-tree 插件
- `--dump-config` / `--dump-default-config` 打印配置树后退出
- launcher 只解析自己开头的 flags，其余透传给被启动的 app

### 测试

CLI 测试位于 `apps/cli/tests/`；涉及 Loader 启动的用 `@deepseek-ai/dsh-loader-smoke`。相关 e2e 场景见 `examples/` 各叶子的 keyless 冒烟。
