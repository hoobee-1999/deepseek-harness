# DeepSeek Harness 文档

面向不熟悉本技术栈的读者。本套文档把 DeepSeek Harness 代码仓库梳理为可读的教程：从"这是什么"到"怎么跑起来"再到"怎么扩展它"。仓库自带的权威文档在 `docs/`，本目录是速览与入门。

**快速链接**: [架构](./ARCHITECTURE.md) | [接口](./INTERFACES.md) | [开发者指南](./DEVELOPER_GUIDE.md)

---

## 核心文档

### [架构](./ARCHITECTURE.md)
系统设计、技术栈、组件结构、启动组合与事件系统。从这里开始了解系统如何运作。

### [接口](./INTERFACES.md)
CLI 命令、Web API、SDK 协议、ACP 协议、插件扩展点与配置接口。集成或使用此系统的参考。

### [开发者指南](./DEVELOPER_GUIDE.md)
环境搭建、开发工作流、编码规范与常见任务。贡献者必读。

---

## 模块

| 模块 | 描述 | README |
|------|------|--------|
| `apps/cli` | `dsh` 命令行入口：profile 启动、插件管理、web 别名 | [模块/cli](./模块/cli.md) |
| `packages/core` | 产品 API 主干：session、tools、agent、agent-loop | [模块/core](./模块/core.md) |
| `packages/bundle` | 可安装的 profile patch 层（base / web-app / headless） | [模块/bundle](./模块/bundle.md) |
| `packages/sdk` + `python` | 进程外 JSON-RPC 协议栈与 Python 孪生 | [模块/sdk](./模块/sdk.md) |

---

## 核心概念

理解这些领域概念有助于导航代码库：

| 概念 | 描述 |
|------|------|
| [插件与 Context](./专有概念/插件与Context.md) | Cordis 插件、服务与 ctx 键 |
| [能力缝](./专有概念/能力缝.md) | Service Definition / Provider / Consumer 三角色 |
| [Profile 与 Bundle](./专有概念/Profile与Bundle.md) | 可叠加配置层与组合机制 |
| [会话日志](./专有概念/会话日志.md) | SessionEvent 事件源与模型上下文来源 |

---

## 入门指南

### 项目新人？

按此路径学习：
1. **[架构](./ARCHITECTURE.md)** - 了解全局
2. **[核心概念](#核心概念)** - 学习领域术语
3. **[开发者指南](./DEVELOPER_GUIDE.md)** - 搭建环境
4. **[接口](./INTERFACES.md)** - 探索公开 API

### 需要集成？

1. **[接口](./INTERFACES.md)** - SDK/ACP/Web API 契约
2. **[架构](./ARCHITECTURE.md)** - 系统边界与数据流

### 首次贡献？

1. **[开发者指南](./DEVELOPER_GUIDE.md)** - 搭建和工作流
2. **[架构](./ARCHITECTURE.md#新的行为放在哪)** - 扩展点映射
3. **[能力缝](./专有概念/能力缝.md)** - 理解可替换能力

---

## 快速参考

### 命令

```bash
pnpm install                # 安装依赖（node ^22.19 || >=24，pnpm 11.7.0）
pnpm run typecheck          # 类型检查
pnpm run build              # 完整构建
pnpm dsh web                # 启动 Web UI（默认 127.0.0.1:3080）
pnpm run test               # 单元测试
pnpm run test:coverage      # CI 覆盖率门禁
```

### 重要文件

| 文件 | 目的 |
|------|------|
| `apps/cli/src/bin.ts` | `dsh` 命令入口 |
| `packages/boot/app-boot/src/index.ts` | 插件树启动序列 |
| `packages/bundle/base/cordis.patch.yml` | 所有 profile 共享的第一配置层 |
| `examples/headless-agent/cordis.yml` | 叶子配置示例 |
| `docs/architecture.md` | 仓库权威架构文档 |
| `AGENTS.md` | 仓库约定（agent 与贡献者必读） |
