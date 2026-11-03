# 04 纯度门禁：值导入 vs 类型导入

## 规则

客户端 bundle 里，**任何 `@deepseek-ai/*` 的"值"导入都会被构建期拒绝**，除非它属于：

1. 宿主基线模块表（`PLATFORM_MODULES`，如 `react`、`react/jsx-runtime`、`@deepseek-ai/cordis`）
2. 本包在 `dsh.client.external` 里显式声明的
3. 白名单里的"可内联线格式层"（`INLINE_SAFE`）
4. 生成的 `/remote` 贡献

否则直接抛：

```
client bundle purity: "<specifier>" is not in the default client externals or <id>'s dsh.client.external,
an inline-safe wire layer, or a generated /remote contribution — cross-plugin value imports are forbidden
```

## 类型导入是安全的

```ts
import type { Context } from '@deepseek-ai/cordis'                 // ✅ 编译后消失
import type {} from '@deepseek-ai/dsh-client-ui-renderer/client'   // ✅ 只为拉 Context 类型合并
import { useState } from 'react'                                   // ✅ react 在基线里
import { Button } from '@deepseek-ai/dsh-client-ui-primitives'     // ❌ 值导入，构建失败
```

`import type` 在打包前被完全抹除，**根本到不了门禁**。

## 为什么这么严

跨插件值导入会把对方的一份运行时实例复制进你的 bundle，或者要求一个模块表给不了的 specifier。跨插件协作只有两条路：

- 行为走 **Cordis 服务**（`export const inject = [...]`）
- UI 走 **座位**（`ctx.slots.register`）

## 自查

构建能过就说明没违规——门禁是构建期 `resolveId` 抛错，跑在产 `lib/client.js` 之前。

验证产物：

```sh
grep -oE 'require\("[^"]+"\)' packages/experimental/hello-page/lib/client.js | sort -u
```

正常应该只剩基线模块：

```
require("react")
require("react/jsx-runtime")
```

## 常见误判

- **`ctx.slots` 的类型从哪来？** 来自 `import type {} from '@deepseek-ai/dsh-client-ui-renderer/client'` 的 Context 类型合并。必须有这一行，否则 `ctx.slots` 不存在。
- **`react` 要不要写进 `dsh.client.external`？** 不要。基线模块对每个动态 bundle 隐式生效，重复声明会被 `verify-client-packages` 报"重复基线模块"。
