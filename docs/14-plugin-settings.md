# 14 插件设置：把 Config 字段变成 Plugins 页的可编辑表单

## 现象

插件有几个"部署相关"的值（aria2 地址、来源筛选、分类）。最常见的错误做法是：在面板里画一个表单，把值存到自己的 JSON 文件，然后再读回来。这样做表面能跑，实际会立刻遇到三个问题：

1. profile 的 `cordis.yml` / `cordis.patch.yml` 里已经有一份值，JSON 里又有另一份，优先级说不清；
2. 改了值之后插件不重启就不生效，因为 `apply(ctx, config)` 拿到的还是启动时的普通字符串；
3. DSH 的设置页、配置目录、重启恢复都看不到这份配置。

## 结论

不要再造一套配置存储。DSH 已经有完整通道：

> **`Config` 里标了 `.volatile()` 的字段 → `settings` 服务投影成表单 → 写入当前 profile 的 `cordis.patch.yml` → Loader 原地更新引用，插件不重启。**

插件要做的只有两件事：

- 把想让人改的字段标成 `.volatile()`，并在每次操作时用 `config.field.get()` 读当前值；
- 如果这个插件是安装进 profile 的 bundle，就在客户端注册 `plugins.row.config`，用 Plugins 页传下来的 `form` 渲染自己的控件。

本文基于 DSH `0.2.2-rc.1`（`fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159`）和 `dsh-downloader` 0.6.0 的实测。相关源码：

- `packages/settings/settings/src/index.ts`：`describe()` / `update()` / `mutate()`、命名空间与 revision
- `packages/settings/settings/src/schema.ts`：`volatileForm()`，决定哪些字段可编辑
- `packages/boot/config-editor/src/index.ts`：校验完整 Config、原子写 profile patch
- `vendor/loader/src/config/entry.ts`：volatile-only 变更原地提交，并发 `loader/volatile-update`
- `packages/client/ui-settings/src/client/config-form.ts`：`ctx.configForms`，客户端读值与写的队列
- `packages/client/ui-plugin-manager/src/client/config-ledger.ts`：`plugins.row.config` 的 key 规则
- `docs/cookbook/adding-a-settings-card.md`：官方的最小示例（但 `.check()` 一节与当前源码不符，见坑 3）

## 机制

### 1. 只有 volatile 字段会出现在表单里

普通字段永不暴露，也不会因为表单写而被重载。Volatile 字段必须在固定路径上（对象/数组的叶子可以，dict/array 内部动态 key 不行）。

```ts
import type { Volatile } from '@deepseek-ai/cordis'
import z from '@deepseek-ai/schemastery'

export interface Config {
  endpoint: Volatile<string>
  qualityFilter: Volatile<'0' | '1' | '2' | undefined>
  category: Volatile<string | undefined>
  tokenRef: string   // 普通字段：不进表单
}

export const Config = z.object({
  endpoint: z.string().pattern(/^https?:\/\/[^@/?#\s]+(?:\/[^\s]*)?$/).default('http://127.0.0.1:8082/jsonrpc').volatile(),
  qualityFilter: z.union(['0', '1', '2']).volatile(),
  category: z.union(['0_0', '1_0']).volatile(),
  tokenRef: z.string().default('ARIA2_TOKEN'),
})
```

### 2. 命名空间是 profile 条目的局部 id

`settings.describe()` 的 `ns` 来自 `entry.options.id`，不是 Loader 的完整 entry id。

实测同一个插件出现两个名字：

```
Config inspect: id = include:downloader, patchId = downloader
settings ns:     downloader
```

所以**不要**硬编码 `include:` 前缀，也不要从 Loader 全名拼接。如果想在 Host 里拿自己的 ns，用 `ctx.fiber.entry?.options.id`（plugin-market 就是这么做的）。

### 3. 写入落到 profile patch

`ctx.configForms.get(ns).mutate(ops, revision)` 最终走 `settings.update/replace/mutate` → `configEditor.edit()`：

- 先解析出完整候选 Config 并校验，失败则文件与运行值都不变；
- 检查 revision，过期写入被拒绝（`settings/conflict`）；
- 原子替换 profile patch（本机 `~/.dsh/profiles/web/cordis.patch.yml`），保留注释与 `!!js`；
- 只对 volatile 字段走原地更新；普通字段的变更仍会走重载。

实测保存 `f=1` 后 patch 里只有这一条，没有第二份 JSON：

```yaml
- id: downloader
  name: "@deepseek-ai/dsh-experimental-downloader"
  config:
    qualityFilter: "1"
```

### 4. 不重启插件

Loader 检测到"只有 volatile 值变了"时，会把新快照提交进原来的 `Volatile` 引用，并在插件 fiber 上发 `loader/volatile-update`。`apply` 不会被重新调用。所以：

- 读值必须 `config.endpoint.get()`（在每次 RPC/每次请求前），不能在构造时缓存字符串；
- 需要重建注册面（例如换地址后重建连接池）时，监听 `loader/volatile-update`，但通常不需要。

### 5. bundle 的 UI 入口是 `plugins.row.config`

Plugins 页按 bundle patch 里的行 id 找配置页，key 固定为 `<包名>#<行 id>`：

```ts
import type {} from '@deepseek-ai/dsh-client-ui-plugin-manager/client'

const ROW_KEY = '@deepseek-ai/dsh-experimental-downloader#downloader'
ctx.slots.inject('plugins.row.config', () => ctx.slots.register({
  name: 'plugins.row.config',
  key: ROW_KEY,
  locale: NS,
  inject: () => ({ t, download }),
}, SettingsPage))
```

页面打开该行时会把绑定好的 `form` 传进来（`ConfigPageForm`：`state` + `mutate`），它底层就是 `ctx.configForms`。**直接用它**，不要自己找命名空间、也不要另建 `ctx.remote.settings` 调用。

```tsx
import type { PluginConfigViewProps } from '@deepseek-ai/dsh-client-ui-plugin-manager/client'

export function SettingsPage({ view, form, t, download }: PluginConfigViewProps & {
  t: Translate
  download: DownloadNamespace
}) {
  // summary: 行的一行说明；page: 表单
  // form.state = { status, value, base, user, revision, writable }
  // form.mutate([{ op: 'set', path: ['qualityFilter'], value: '1' }], form.state.revision)
}
```

### 6. 表单行为约定

- 只提交改动过的字段（`{ op: 'set', path: [field], value }`）；未改的不写，避免把 schema 默认值钉进 profile。
- 取消 = 丢弃本地草稿，不调用 `mutate`；不要用"保存前先清空"。
- `mutate` 返回 `false` 表示 Host 拒绝（校验/冲突/只读），`throw` 表示传输失败；两种都要显示可重试错误并保留旧有效值。
- 需要清空字段用 `{ op: 'unset', path: [field] }`；不要全量 `replace`，那会把未返回给客户端的 secret 一起重置。
- 表单里的"测试连接"应该单独走一个 Host 方法（例如只调 `aria2.getVersion`），不要为了测试而保存配置，也不要在测试路径里顺手提交任务。

## 坑位

### 坑 1：命名空间写成 `include:<id>`

**现象**：Plugins 页行存在，但打开配置页是空的；`settings.describe()` 里找不到这个 ns。

**原因**：`ns` 是 `entry.options.id`（局部 id），不是 Loader 的 `include:` 全名。行 key 也必须用 bundle patch 里的行 id。

**修法**：行 key 用 `<包名>#<patch 行 id>`；Host 侧需要 ns 时用 `ctx.fiber.entry?.options.id`；不要让客户端猜。

### 坑 2：共享类型把 host-only 模块拉进客户端 tsconfig

**现象**：`tsc -b` 报 `TS6307: File .../src/aria2.ts is not listed within the file list of project .../tsconfig.client.json`，或者 `TS2591: Cannot find name 'Buffer'`。

**原因**：客户端的 `tsconfig.client.json` 只 include 明确列出的文件。共享的 `src/types.ts` 如果 import 了某个会再 import `node:fs` / `Buffer` 的 host 模块，编译图就把整个 host 链拉进来了。

**修法**：把纯类型/常量放到无 Node 依赖的模块（例如 `catalog.ts`），让 `types.ts` 只从那里取；需要客户端用的常量模块显式加进 `tsconfig.client.json` 的 `include`。`settings.ts` 这种会 import `aria2.ts` 的模块只留给 Host。

### 坑 3：照抄 cookbook 的 `.check()`

**现象**：`Config.check(...)` 报 `Property 'check' does not exist`。

**原因**：`docs/cookbook/adding-a-settings-card.md` 提到用 `.check()` 做跨字段校验，但当前 vendor 的 schemastery 3.19 没有实现这个方法。文档比代码新（或比这份 checkout 新）。

**修法**：用 `z.union([...])` / `.pattern()` 做字段校验；跨字段规则放在解析函数里（例如从旧 `feedUrl` 推导默认值），并在使用点保证非法值不会进入请求。不要改 vendor 去补 API。

### 坑 4：以为有通用表单渲染器

**现象**：找不到"根据 schema 自动生成 select/文本/密码框"的页面。

**原因**：`autoGenerate` 目前没有客户端消费者；`SettingsForm` / `SettingsValueField` 只有文本/数字；`ConfigField` 没有作为公共导出。

**修法**：需要下拉、测试按钮、自定义校验时，自己写控件，但**传输仍然用页面传入的 `form`**。不要因为控件是自定义的就把配置另存一份。

### 坑 5：overlay 开发看不到设置页

**现象**：`pnpm dsh web --patch ...overlay.yml` 加载的插件，Plugins 页里没有它，也没有配置入口。

**原因**：Plugins 页的包/行来自 profile 已安装的 bundle；overlay 插入的是实验行，不是 package。

**修法**：验证设置页时，把构建产物临时装进 profile：

```sh
dsh plugin --profile web add link:/path/to/plugin/bundle
# 或 plugin_manager install_bundle link:/path/to/plugin/bundle
```

验完卸载。**实测卸载后 profile patch 里会留下孤立的 `- id: downloader` override，`node_modules` 里也可能留下链接**；要手工清掉（见坑 6）。

### 坑 6：卸载残留

**现象**：`remove_bundle` 后 `package.json` 已恢复，但 `cordis.patch.yml` 里还有：

```yaml
- id: downloader
  name: "@deepseek-ai/dsh-experimental-downloader"
  config:
    qualityFilter: "2"
```

**原因**：配置 override 是 profile patch 的独立行，不随包卸载自动删除。

**修法**：验收结束前把这条删除，并检查 `node_modules/@deepseek-ai/<包>` 是否还有失效链接；否则下次启动 Loader 会带着一个指向不存在的 id 的覆盖。

### 坑 7：构造时缓存配置值

**现象**：保存设置后，下一次请求仍用旧地址/旧筛选。

**原因**：在 `apply` 或构造函数里把 `config.endpoint` 存成了字符串。volatile 更新只改引用，不会重新调用 `apply`。

**修法**：每次操作时 `.get()`；需要一致快照时，一次操作内取一次并传下去。

### 坑 8：端点可编辑 = 凭据可下发

**现象**：为了让"测试连接"能跑，把 token 传给浏览器或塞进 URL。

**原因**：地址和凭据是两件事。

**修法**：地址可以经 Remote 返回给 Client；token 只在 Host 用 `ctx.credentials` 解析，测试连接由 Host 用当前 token 发起，返回分类后的结果（成功/鉴权失败/超时/不可达/协议错误），不返回 token 或服务器原文。

## 验证清单

```sh
# 构建产物（Typert + bundle）
pnpm --filter @deepseek-ai/dsh-experimental-downloader run bundle
./node_modules/.bin/tsc -b packages/experimental/dsh-downloader
pnpm vitest run packages/experimental/dsh-downloader/tests/
```

浏览器（临时安装后）：

1. Plugins → 插件包 → 行 → 配置，确认表单出现且值为当前有效值；
2. 改一个字段保存，确认 profile patch 只多出该字段，且没有第二份 JSON；
3. 重新打开页面，确认值来自 patch；
4. 非法值（错误 URL、不支持的分类）确认就地报错且保存禁用；
5. 取消确认不写；
6. 测试连接确认不保存、不添加下载任务；
7. 卸载后确认 profile 恢复（含坑 6 的残留）。

## 实测记录（2026-11-14）

`dsh-downloader` 0.6.0，web profile 临时 `link:` 安装：

- 设置页初始值 `f=2`、`c=0_0`、`endpoint=http://10.0.0.7:8082/jsonrpc`；
- 保存 `f=1` 后 patch 写入 `qualityFilter: "1"`；
- `ftp://not-a-url` 触发 `请输入不含凭据的完整 http(s) 地址。`，保存按钮禁用；
- 测试连接返回“服务器未按 aria2 协议响应。”；
- 卸载并清理孤立 override 后，profile 的 `package.json` 与 `cordis.patch.yml` 恢复原状。

## 参考

- [settings 子系统](https://github.com/deepseek-ai/deepseek-harness/blob/fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159/docs/subsystems/settings.md)
- [config-editor](https://github.com/deepseek-ai/deepseek-harness/blob/fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159/packages/boot/config-editor/README.md)
- [volatile 配置](https://github.com/deepseek-ai/deepseek-harness/blob/fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159/vendor/loader/README.md)
- [live configuration forms cookbook](https://github.com/deepseek-ai/deepseek-harness/blob/fcc0ad462172522aaf5c93a7b2e01f6b9f0b0159/docs/cookbook/adding-a-settings-card.md)
- [本仓库 12 篇：装进桌面版](12-install-to-desktop.md)
- [本仓库 03 篇：座位注册](03-slots.md)
