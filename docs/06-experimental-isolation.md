# 06 实验包隔离：不能进正式 bundle

## 规则

`packages/experimental/*` 下的包**不能被任何正式组合挂载**。

默认产品隔离门禁会检查：正式 bundle 的 `package.json` 依赖、运行时导入、组合配置里都不能出现实验包。

```
verify-default-product-isolation: 302 packages, 2276 runtime sources, 10 configurations,
306 composed Web plugins exclude experimental packages.
```

## 那它怎么加载？

靠 **测试用 overlay 覆盖层**。仓库里的先例：

```
apps/web/tests/client-ui-stats-panel.overlay.yml
```

内容极简：

```yaml
# Test-only composition; shipped compositions do not mount experimental packages.
- insert:
    - id: hello-page
      name: '@deepseek-ai/dsh-experimental-hello-page'
```

## 启动

```sh
pnpm dsh web --patch apps/web/tests/hello-page.overlay.yml
```

`--patch` 是**启动器参数**（`dsh` 的），不在 `dsh web --help` 里，容易找不到。

## 常见错误

| 错误做法 | 后果 |
|---|---|
| 往 `packages/bundle/web-app/cordis.patch.yml` 里加行 | 隔离门禁红 |
| 往 `packages/bundle/web-app/package.json` 加依赖 | 隔离门禁红 |
| 以为 `cordis.source.patch.yml` 能让页面出现 | 它只插 Host 半边，页面是客户端 bundle 的事 |

## 记住

- `cordis.source.patch.yml` = 源码模式下的 Host 插件行，插的是 `./src/index.ts`
- overlay = 正式组合之上叠加的测试行，插的是包名，会加载 `dsh.client` 声明的浏览器半边

两者不是一回事，一个插件通常两个都要（前者给源码调试，后者给界面加载）。
