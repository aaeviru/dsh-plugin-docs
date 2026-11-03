# 05 文案与元数据：两道看不见的关

## 第一道：UI 文案必须走 locale 字典

### 现象

```
verify-client-ui-i18n: 5 hard-coded UI string(s):
  src/client/index.tsx:15:73 JSX text: "Hello Page"
  src/client/index.tsx:16:64 JSX text: "A simple page with a counter."
  src/client/index.tsx:22:9  JSX text: "Increment"
  src/client/index.tsx:24:29 JSX text: "Count:"
  src/client/index.tsx:44:68 label property: "Hello"
```

### 修法

1. 建 `src/client/locales.ts`：

```ts
export const en = {
  title: 'Hello page',
  description: 'A simple page with a counter.',
  increment: 'Increment',
  count: 'Count',
  panel: 'Hello',
} as const

export const zh: Record<keyof typeof en, string> = {
  title: 'Hello 页面',
  description: '一个带计数器的简单页面。',
  increment: '增加',
  count: '计数',
  panel: 'Hello',
}
```

2. 入口里声明命名空间并注册：

```ts
declare module '@deepseek-ai/dsh-client-ui-slots' {
  interface LocaleNamespaceMap {
    helloPage: keyof typeof en
  }
}

export function apply(ctx: Context): void {
  ctx.effect(() => ctx.locale.register('helloPage', { en, zh }))
  const t = ctx.locale.bind('helloPage')
  // 之后注册座位时用 t(...)
}
```

3. 座位注册带 `locale: 'helloPage'`，`label` 用 `() => t('panel')`。

### 注意

- `label` 必须传**函数**。传字符串不会被类型拦住，但会被门禁拦住。
- 文案通过 `inject` 以普通字符串传给组件，组件不接触 `ctx`。

## 第二道：包元数据要八语言

### 现象

```
packages/experimental/hello-page/locale: ENOENT: no such file or directory, open '.../locale/en.json'
```

### 规则

只要包出现在配置目录里，就必须有 `locale/` 且覆盖 `SHIPPED_LOCALE_IDS` 的**全部语言**（de / en / es / fr / ja / pt / ru / zh）。

### 最小内容

`locale/en.json`：

```json
{
  "meta": {
    "title": "Hello page",
    "description": "Open the Hello page from the left sidebar and increment a counter for Web testing."
  }
}
```

每个语言只有这两个键，键集合必须一致。门禁会检查：

- `meta.description` 以句末标点结尾
- 非英语语言不能直接照抄英文或中文原文（会被判"未翻译"）
- 占位符/字面 token 不能变

### 别漏 exports

```json
"exports": {
  "./locale/*.json": "./locale/*.json"
},
"files": ["locale/*.json", "lib/index.js", "lib/client.js", "lib/types/**/*.d.ts"]
```

## README 的四个必备部分

```markdown
---
description: "一句话描述"
kind: "package-reference"
---

# 包名

## Summary

一段话说明这个包做什么（≤100 词）。

## Model Experience

None, as ...（不注册工具/提示词/模型输入）

#### KV Cache effect

No effect. ...
```

外加：

```markdown
## Known Limitations and Deferred Work
```

缺任何一节对应门禁都会红：

| 章节 | 门禁 |
|---|---|
| `## Summary` | `verify-package-readme-summaries` |
| `## Model Experience` + KV Cache | `verify-package-readme-model-experience` |
| `## Known Limitations and Deferred Work` | `verify-package-readme-limitations` |

补充：如果 Model Experience 真的只有一句话，可以往 `scripts/verify-package-readme-model-experience.ts` 的 `SENTENCE_MODEL_EXPERIENCE` 表里加一条（和 `client-ui-stats` 同款）。**这是脚本自带的声明机制，但确实是改共享门禁文件，评审时要说明。**
