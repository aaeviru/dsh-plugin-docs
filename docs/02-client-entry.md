# 02 客户端入口：为什么不能叫 `index.tsx`

## 现象

包内执行 bundle：

```sh
pnpm --filter @deepseek-ai/dsh-experimental-hello-page run bundle
```

报：

```
[UNRESOLVED_ENTRY] Cannot resolve entry module src/client/index.ts.
```

可文件明明存在，只是叫 `index.tsx`。

## 原因

打包预设 `clientBundle` 在**没有指定构建面**（`DSH_BUILD_FACE` 未设置，也就是包内单独跑 `tsdown`）时，入口是**写死的字面量**：

```ts
const clientEntry = face === undefined ? 'src/client/index.ts' : 'lib/types/client/index.js'
```

它不会做扩展名回退，所以 `.tsx` 找不到。

而仓库整体构建时（`DSH_BUILD_FACE=client`）走的是 `lib/types/client/index.js`，那是 tsc 从 `src/client/index.tsx` 编译出来的，**所以全量构建能过、包内 bundle 会挂**——这个不一致非常隐蔽。

顺带一个证据：整个仓库 `packages/**/src/client/index.tsx` 一个都没有。

## 正确做法

入口保持**无 JSX**，组件拆到独立文件：

```
src/client/
  index.ts        ← 只有 inject / apply，不含 JSX
  HelloPage.tsx   ← 页面组件
  HelloIcon.tsx   ← 图标组件
  locales.ts      ← 文案字典
```

`index.ts` 里这样写：

```ts
import type { Context } from '@deepseek-ai/cordis'
import type {} from '@deepseek-ai/dsh-client-ui-renderer/client'
import type {} from '@deepseek-ai/dsh-client-ui-layout/client'
import type {} from '@deepseek-ai/dsh-client-ui-sidebar/client'
import { HelloPage } from './HelloPage.tsx'
import { HelloIcon } from './HelloIcon.tsx'

export const inject = ['slots', 'locale']

export function apply(ctx: Context): void {
  ctx.slots.inject('main', () => ctx.slots.register({ name: 'main', key: 'hello-page' }, HelloPage))
  ctx.slots.inject('sidebar.panellist', () => ctx.slots.register({
    name: 'sidebar.panellist', id: 'hello-page', order: 30,
  }, HelloIcon))
}
```

## 自查

```sh
pnpm --filter <包名> run bundle
```

退出码 0 且产出 `lib/client.js` 才算过。**不要只看全量构建通过就以为没事。**
