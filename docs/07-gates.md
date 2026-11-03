# 07 门禁清单：按这个顺序跑

## 顺序很重要

先包内构建，再单包门禁，最后聚合。**别一上来跑全量**，很慢且报错难定位。

## 1. 包内构建（最快，最先跑）

```sh
pnpm --filter @deepseek-ai/dsh-experimental-hello-page run bundle
```

这一步能抓到 `UNRESOLVED_ENTRY` 这类入口问题。**全量构建过了不代表这步能过**（入口路径两条分支不同）。

## 2. 单包类型检查

```sh
pnpm exec tsc -b packages/experimental/hello-page/tsconfig.json --pretty false
```

## 3. 定向 lint

```sh
pnpm exec oxlint -c .oxlintrc.json packages/experimental/hello-page
```

仓库用 **oxlint**，不是 eslint（没有 `eslint.config.js`，直接跑 eslint 会报找不到配置）。

## 4. 元数据类门禁

```sh
pnpm exec tsx scripts/verify-locale-metadata.ts
pnpm exec tsx scripts/verify-package-readme-summaries.ts
pnpm exec tsx scripts/verify-package-readme-model-experience.ts
pnpm exec tsx scripts/verify-package-readme-limitations.ts
pnpm exec tsx scripts/verify-npm-install-layout.ts
pnpm exec tsx scripts/verify-dsh-package-licenses.ts
```

## 5. 包与策略门禁

```sh
pnpm exec tsx scripts/verify-client-packages.ts
pnpm exec tsx scripts/verify-client-ui-i18n.ts
pnpm exec tsx scripts/verify-package-dependencies.ts
pnpm exec tsx scripts/verify-default-product-isolation.ts
pnpm exec tsx scripts/verify-package-meta.ts
pnpm exec tsx scripts/verify-export-jsdoc.ts
pnpm exec tsx scripts/verify-cordis-config.ts
pnpm exec tsx scripts/verify-package-paths.ts
```

## 6. 生成物一致性（新增包/新座位后必跑）

```sh
pnpm exec tsx scripts/gen-tsconfig-paths.ts --check
pnpm exec tsx scripts/gen-plugin-packages.ts --check
pnpm exec tsx scripts/gen-client-catalog.ts --check
pnpm exec tsx scripts/gen-config-catalog.ts --check
pnpm exec tsx scripts/gen-module-graph.ts --check
pnpm exec tsx scripts/gen-dependency-catalog.ts --check
```

报 stale 就用不带 `--check` 的同名脚本重新生成，生成的 diff 要一起提交：

```sh
pnpm run gen-plugin-packages
pnpm run gen-client-catalog
pnpm run gen-config-catalog
pnpm run gen-module-graph
```

## 7. 聚合

```sh
pnpm exec tsc -b tsconfig.client.json     # 70 个包的客户端聚合
pnpm run build:check                       # host + client 两个聚合
```

## 典型报错速查

| 报错 | 对应的漏项 |
|---|---|
| `UNRESOLVED_ENTRY: Cannot resolve entry module src/client/index.ts` | 入口是 `.tsx` |
| `client bundle purity: ...` | 值导入了别的插件 |
| `missing ## Summary` / `## Model Experience` / `## Known Limitations` | README 缺章节 |
| `locale: ENOENT ... en.json` | 没有 locale 元数据 |
| `has no workspace version 0.2.2-alpha.3` | `version` 不是工作区版本 |
| `must declare "license": "MIT"` | 缺 license |
| `cannot be generated`（tsconfig paths） | 包名 ≠ 目录名，要手写别名 |
| `... is stale`（gen-*） | 生成物没重新生成 |
| `hard-coded UI string(s)` | 文案没走 locale |
