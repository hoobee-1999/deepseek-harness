# packages/bundle

可安装的 `dsh --profile` patch 层。每个 bundle 是一个 npm 包，其 `dsh.bundle.patch` 指向 `cordis.patch.yml`；bundle 的实质就是这份补丁。

## 结构

```
packages/bundle/
├── base/          # @deepseek-ai/dsh-base — 所有 profile 共享的第一层
├── web-app/       # @deepseek-ai/dsh-web-app — 浏览器 GUI 表面
└── headless/      # @deepseek-ai/dsh-headless — 一次性无头任务模式
```

## 关键文件

| Bundle | 文件 | 目的 |
|------|------|------|
| base | `packages/bundle/base/cordis.patch.yml` | 451 行的大 `insert`：timer、hmr、llm、session、typert、agent、sandbox/subprocess、tools、web、system-prompt、agent-loop、llm-deepseek、subagent、workflow、todo 等 100+ 行，含中性默认值 |
| web-app | `packages/bundle/web-app/cordis.patch.yml` | webserver（默认 127.0.0.1:3080）、前端静态 dist、浏览器 client 插件 roster；大范围 disable base 里的 per-agent 工具行 |
| web-app | `packages/bundle/web-app/src/startup.ts` | `webStartup` provider：解析 `--host/--port/--no-open/--trusted-host` |
| headless | `packages/bundle/headless/src/startup.ts` | `headlessStartup` provider：解析 `[task...]` 位置参数 |
| headless | `packages/bundle/headless/src/index.ts` | `headless-runner` 插件：创建 Agent、驱动到 quiescence、打印最终文本后以 0/1 退出 |

## 依赖

**本模块依赖**：
- `packages/boot/app-boot` — profile/bundle 解析
- 产品包全集（base 挂载几乎所有能力）

**依赖本模块的**：
- `apps/cli` — `dsh` 命令按 profile 模板引用这些 bundle
- 用户自定义 profile（`dsh.profile.bundles` 列表可引用任意 dsh bundle）

## 规范

### patch 层策略

- 行内某值随模式不同就不放 base，交给各模式 bundle 完整重述
- 同一行只允许"一个 bundle 层 + 用户层"写值
- base 用 `!!js` 表达式示例：`root: !!js dshHomePath('sessions')`、`disabled: !!js process.platform === 'win32'`（bash/pwsh 平台互斥）

### 平台互斥

bash 与 pwsh 的工具行在 `!!js process.platform` 上互斥禁用，保证 Windows/非 Windows 只挂一种后端。

### 测试

bundle 层行为经 `examples/` 的 keyless 冒烟与真实 cordis.yml 装载测试验证（`dsh-loader-smoke`）。改 bundle 的 patch 会影响所有 profile，需要构建后验证。
