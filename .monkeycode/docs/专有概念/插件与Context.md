# 插件与 Context

插件是 DeepSeek Harness 一切功能的最小组成单元：模型适配器、工具、持久化、权限、甚至 Agent 主循环本身都是插件。

## 什么是插件？

插件是实现了 Cordis `Service` 的对象。两种形态：

1. **函数插件**（最常用）：带可选 `inject` 与 `apply(ctx)` 字段的函数。贡献通过 `ctx.effect()`、`ctx.on()`、`ctx.waterfall()` 完成。named-export `name` / `inject` / `Config` / `apply`，**没有** default export。
2. **服务类**：继承 Cordis `Service` 的类，生命周期由 Cordis 挂载进当前 context。服务包 default-export 这个类。

**关键特征**：
- 插件之间不 import 具体实现，而是通过 `ctx.<key>` 找服务
- 注册即副作用，且注册返回 disposer——插件卸载时全部可逆地撤销
- 依赖通过 `inject` 声明式表达，加载顺序由服务可用性驱动，而非手动 boot 排序

## 什么是 Context？

Context（`ctx`）是服务的仓库。一个服务声明一个稳定的 `ctx.<key>`（如 `ctx.tools`、`ctx.llm`、`ctx.sessions`），其他插件按 key 找到它。服务注册进 context 通过：

```ts
import { Service } from '@deepseek-ai/cordis'

export abstract class FileSystem extends Service {
  constructor(ctx: Context) {
    super(ctx, 'fs')
  }
}
```

服务与类型之间通过 TypeScript 声明合并绑定：

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    fs: FileSystem
  }
}
```

可选服务用 `ctx.get(name)` 读取（严格读全局服务 store）；`ctx.<name>` 保留给已声明的注入。

## 代码位置

| 方面 | 位置 |
|------|------|
| Cordis 源码 | `vendor/cordis/` |
| 插件规则 | `packages/AGENTS.md` |
| Cordis 概念入门 | `docs/cordis-primer.md` |
| 新增包 cookbook | `docs/cookbook/adding-a-package.md` |

## 添加一个新插件包

1. 加入已有 group：`packages/<group>/<pkg>/`
2. 函数插件 named-export 四件套；服务包 default-export service 类
3. 注册到恰一个 aggregate（Host 或 Client tsconfig）
4. 测试放 `tests/`，`src/types.ts` 只放类型
5. 写 README 与 `./invariant`

## 不变量

1. **注册即副作用**：一切贡献走 `ctx.effect()` / `ctx.on()` / `ctx.waterfall()`，注册返回 disposer；HMR/卸载时按注册逆序撤销。
2. **不混合导出形式**：函数插件没有 default export，服务包 default-export 服务类。混合会让 Loader 丢弃函数插件的命名空间。
3. **可选服务用 `ctx.get`**：属性代理是拓扑敏感的，`ctx.get` 严格读全局服务 store。
