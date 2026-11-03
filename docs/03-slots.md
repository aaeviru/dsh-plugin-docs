# 03 座位注册：main 与 sidebar.panellist

## 两个座位

一个"左侧栏图标 + 中间主面板页面"的插件，占两个座位：

| 座位 | 作用 | 关键字段 |
|---|---|---|
| `main` | 中间主面板的页面 | `key` |
| `sidebar.panellist` | 左侧栏图标列表 | `id`、`order`、`label` |

注意 `main` 用 `key`，`sidebar.panellist` 用 `id`——**不是同一个字段名**。

## 注册写法

```ts
ctx.slots.inject('main', () => ctx.slots.register({
  name: 'main',
  key: 'hello-page',
}, HelloPage))

ctx.slots.inject('sidebar.panellist', () => ctx.slots.register({
  name: 'sidebar.panellist',
  id: 'hello-page',
  order: 30,
  label: () => t('panel'),
  locale: 'helloPage',
}, HelloIcon))
```

要点：

- 用 `ctx.slots.inject(座位名, () => ctx.slots.register(...))`，**不是**裸 `register`。前者会等座位被声明、座位撤销时自动移除、随插件 fiber 一起销毁。
- `label` 是**函数**（`() => t('panel')`），不是字符串——字符串会触发 i18n 门禁。
- 图标组件接收 `{ size, active }`：

```tsx
function HelloIcon({ size, active }: { size: number; active: boolean }) {
  return <svg width={size} height={size} viewBox="0 0 24 24" aria-hidden="true">...</svg>
}
```

## inject 的两个不同含义（很容易混）

| 位置 | 含义 | 取值 |
|---|---|---|
| 客户端入口 `export const inject` | **Cordis 服务依赖** | `['slots', 'locale']` |
| `package.json` 的 `dsh.client.inject` | **包名依赖边（仅信息性）** | 如 `["@deepseek-ai/dsh-client-locale"]` |

两者不能互换。`dsh.client.inject` **不参与排序、也不换取模块外部化**——它只是给预检和 HMR diff 看的元数据。

真正的模块外部化由 `dsh.client.external` 决定（见 [04 纯度门禁](04-purity-gate.md)）。

## 组件不要碰 ctx

组件只通过注册时传的 props / `inject` 拿数据。需要文案就通过 `inject` 传进去：

```ts
ctx.slots.register({
  name: 'main', key: 'hello-page', locale: 'helloPage',
  inject: () => ({ title: t('title'), description: t('description') }),
}, HelloPage)
```

组件里只接收 `{ title, description }`，不引用 `ctx`。
