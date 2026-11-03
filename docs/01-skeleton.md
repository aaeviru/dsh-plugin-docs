# 01 包骨架：一个能构建的插件包需要什么

## 最容易踩的坑

> **只写 `package.json` + `tsdown.config.ts` + `src/` 是构建不出来的。**

构建系统的客户端入口是 **tsc 产物**（`lib/types/client/index.js`），不是源码。没有 `tsconfig.json` 就没有 `lib/types`，构建会去找一个永远不存在的入口文件。

## 完整文件清单

以 `packages/experimental/hello-page/` 为例：

| 文件 | 作用 | 缺了会怎样 |
|---|---|---|
| `package.json` | 包名、导出、`dsh.client` 清单 | 构建读不到包名，直接报错 |
| `tsconfig.json` | 产出 `lib/types`（客户端入口的来源） | **入口找不到，构建失败** |
| `tsdown.config.ts` | 复用 `clientBundle` 预设 | 不产 `lib/client.js` |
| `cordis.source.patch.yml` | 源码覆盖层，插入 Host 半边 | 源码模式加载不到 |
| `src/index.ts` | Host 半边（可以是空壳） | 加载器缺 Host 入口 |
| `src/client/index.ts` | 浏览器半边，注册座位 | 界面里什么都没有 |
| `src/client/*.tsx` | 组件 | — |
| `src/client/locales.ts` | 文案字典 | i18n 门禁失败 |
| `locale/*.json` | 包元数据翻译（八语言） | locale 门禁失败 |
| `README.md` | 含必备章节 | README 门禁失败 |
| 聚合登记 | `tsconfig.client.json` 加一行 | tsc 不编译这个包 |

## 最小 `tsconfig.json`

```json
{
  "extends": "../../../tsconfig.base.client.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "lib/types"
  },
  "include": ["src"],
  "references": [
    { "path": "../../../vendor/cordis" },
    { "path": "../../client/ui-renderer" },
    { "path": "../../client/ui-layout" },
    { "path": "../../client/ui-sidebar" }
  ]
}
```

`references` 只列**本包真正用到的**依赖。别照抄大包那十几条，会带进无关的编译依赖。

## 最小 `tsdown.config.ts`

```ts
import { clientBundle } from '../../client/tsdown.client.ts'

export default clientBundle(import.meta.dirname, ['lib/types/index.js'])
```

就这两行。

## 聚合登记

在仓库根的 `tsconfig.client.json` 里加一行（按字母序放）：

```json
{ "path": "./packages/experimental/hello-page" }
```

**只加这一行**，别动其它条目。

## 路径别名（包名 ≠ 目录名时必须手写）

如果包名 `@deepseek-ai/dsh-experimental-hello-page` 和目录名 `hello-page` 对不上，自动生成器会跳过它，需要在 `tsconfig.base.json` 的 `paths` 里手写：

```json
"@deepseek-ai/dsh-experimental-hello-page": ["./packages/experimental/hello-page/src"],
"@deepseek-ai/dsh-experimental-hello-page/client": ["./packages/experimental/hello-page/src/client/index.ts"],
```

判定命令：

```sh
pnpm exec tsx scripts/gen-tsconfig-paths.ts --check
```

报 `cannot be generated` 就是缺这个。
