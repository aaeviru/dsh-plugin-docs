# 10 坑位速查

按踩到的顺序编号。每条：**现象 → 原因 → 修法**。

---

### 1. 五个文件建完，构建不出来

**现象**：`lib/` 里没有产物，或构建找不到入口。
**原因**：客户端入口是 tsc 产物 `lib/types/client/index.js`，没有 `tsconfig.json` 就没有它。
**修法**：补 `tsconfig.json` + 在 `tsconfig.client.json` 注册。见 [01](01-skeleton.md)。

---

### 2. `UNRESOLVED_ENTRY: Cannot resolve entry module src/client/index.ts`

**现象**：包内 `pnpm --filter <pkg> run bundle` 失败，但全量构建能过。
**原因**：入口写成 `index.tsx`，而预设的字面量是 `index.ts`，不做扩展名回退。
**修法**：入口改 `index.ts`（无 JSX），组件拆到独立 `.tsx`。见 [02](02-client-entry.md)。

---

### 3. `client bundle purity: ...`

**现象**：构建直接抛错。
**原因**：值导入了别的 `@deepseek-ai/*` 包。
**修法**：改 `import type`；需要行为就走 Cordis 服务，需要 UI 就走座位。见 [04](04-purity-gate.md)。

---

### 4. `verify-client-ui-i18n: N hard-coded UI string(s)`

**现象**：门禁列出所有硬编码文案及行列号。
**原因**：JSX 文本 / `label: '字符串'`。
**修法**：建 `locales.ts`、`ctx.locale.register`、`t()`，`label` 传函数。见 [05](05-locale.md)。

---

### 5. `locale: ENOENT ... en.json`

**现象**：`verify-locale-metadata` 失败。
**原因**：缺 `locale/*.json`，或者语言不全（要八种）。
**修法**：建 `en/zh/de/es/fr/ja/pt/ru.json`，每个只有 `meta.title` + `meta.description`。见 [05](05-locale.md)。

---

### 6. README 三门禁

**现象**：`missing ## Summary` / `missing ## Model Experience` / `missing ## Known Limitations and Deferred Work`。
**修法**：照 `client-ui-stats/README.md` 补章节。见 [05](05-locale.md)。

---

### 7. `has no workspace version 0.2.2-alpha.3`

**现象**：`verify-npm-install-layout` 失败。
**原因**：`version: "0.0.0"`。
**修法**：改成工作区版本（跟其它包一致）。

---

### 8. `must declare "license": "MIT"`

**修法**：`package.json` 加 `"license": "MIT"`。

---

### 9. `cannot be generated`（tsconfig paths）

**现象**：`gen-tsconfig-paths --check` 失败。
**原因**：包名 `dsh-experimental-hello-page` 和目录名 `hello-page` 不一致，生成器跳过。
**修法**：在 `tsconfig.base.json` 的 `paths` 手写两条别名。见 [01](01-skeleton.md)。

---

### 10. `... is stale`

**现象**：`gen-plugin-packages` / `gen-client-catalog` / `gen-config-catalog` / `gen-module-graph` 的 `--check` 失败。
**原因**：新增包或新座位后，生成的目录没更新。
**修法**：跑不带 `--check` 的同名脚本，把 diff 一起提交。见 [07](07-gates.md)。

---

### 11. 实验包加进正式 bundle → 隔离门禁红

**现象**：`verify-default-product-isolation` 失败。
**原因**：往 `packages/bundle/web-app/` 里加了行或依赖。
**修法**：撤掉，改用 `apps/web/tests/*.overlay.yml`。见 [06](06-experimental-isolation.md)。

---

### 12. `dsh web` 报 `EROFS: read-only file system: ~/.dsh/profiles/web/cordis.yml`

**原因**：沙箱只允许写工作区，`~/.dsh` 在外面。
**修法**：`DSH_HOME=/tmp/xxx`（或工作区内任意可写目录）再启动。

---

### 13. Playwright 报 `Executable doesn't exist ... chrome-headless-shell`

**原因**：浏览器没装，或装在沙箱外。
**修法**：`PLAYWRIGHT_BROWSERS_PATH=<工作区>/node_modules/.cache/ms-playwright pnpm exec playwright install chromium`。

---

### 14. chromium 报 `libnspr4.so: cannot open shared object file`

**原因**：缺系统库。
**修法**：`ldd` 列出全部缺失项，一条 `apt-get` 装齐。见 [09](09-verification.md)。

---

### 15. `locator.click()` 超时，但元素可见

**原因**：全屏遮罩挡住，Playwright 只报 timeout。
**修法**：`document.elementFromPoint()` 定位遮挡元素；本次是两个首启弹窗，`Escape` 关掉。见 [09](09-verification.md)。

---

### 16. `pnpm exec eslint` 报找不到配置

**原因**：仓库用 **oxlint**，没有 `eslint.config.js`。
**修法**：`pnpm exec oxlint -c .oxlintrc.json <路径>`。

---

### 17. Codex 会话文件系统只读 / product-error

**现象**：Codex 能读能分析，但写不了文件；或直接 `product-error (stage: turn)`。
**原因**：Codex 是独立进程，工作目录权限和产品可用性由它自己的配置决定，**调用方拿不到也改不了**。
**修法**：只能从 Codex 那一侧配置解决。派活前先确认它能写。

---

---

### 18. overlay 按包名插入报 `ERR_MODULE_NOT_FOUND`

**现象**：`dsh web --patch <overlay>` 启动时提示 `1 entry did not activate`，错误是找不到包，且提示路径指向 `<DSH_HOME>/profiles/web/`。
**原因**：加载器经 tsx 用 `tsconfig.base.json` 的 `paths` 解析包名，而该包没有别名（`gen-tsconfig-paths` 只处理「包名与目录同名」的包）。
**修法**：手写三条别名（包、`/types`、`/client`），见 [06](06-experimental-isolation.md)。

---

### 19. 改包名后报 `TYPERT manifest names package ...`

**现象**：`typert-loader: ... TYPERT manifest names package "old-name" — the manifest must be owned by the package that exports it`，插件不激活。
**原因**：`lib/typert.host.js` / `lib/typert.remote-client.js` 是**生成物**，里面写死了生成时的包名；改 `package.json` 的 `name` 之后没有重新生成。
**修法**：重跑 `pnpm exec tsx scripts/generate-typert.ts`，确认产物里的包名已更新，再重新构建。

---

### 20. 客户端 bundle 报 `client bundle isolation ... experimental input`

**现象**：客户端打包失败，错误指出某个 `lib/typert.remote-client.js` 是 experimental input。
**原因**：隔离门禁按**包名前缀**判断实验身份（`@deepseek-ai/dsh-experimental-*`）。包叫别的名字时会被当成正式包，正式包的客户端 bundle 不允许引用实验目录里的产物。
**修法**：把包名改成带 `dsh-experimental-` 前缀（与 hello-page 一致）；客户端里不要用包名自引 `xxx/types`，改用相对源码路径。

---

---

### 21. 派给 Codex 的活，它经常跑不了门禁

**现象**：`vitest` 报 `EROFS: read-only file system ... node_modules/.vite-temp`；`server.listen(127.0.0.1)` 报 `EPERM`；或整轮直接失败（`category: limit` / `category: service`）。
**原因**：Codex worker 跑在自己的沙箱里 —— harness 检出对它是只读的，本地监听被禁，且有用量/服务上限。
**修法**：任务书里就写明「跑不了就用仓库内等价命令，并如实标注」；**验收必须自己重跑**，它的自述只当参考。需要 stub HTTP 的测试（如 aria2 客户端）只能由验收方跑。

---

### 22. 首启弹窗不是 Escape 能关的

**现象**：Playwright 点击超时，提示 `<div aria-hidden="true" class="_mask_..."> intercepts pointer events`；连按 Escape 无效。
**原因**：挡路的是 **"Preview Notice"**（0.2 预览提示），只有一个 `Continue` 按钮。
**修法**：循环处理 —— 有遮罩时优先点 `Continue | Got it | Close`，没有再按 Escape：

```js
for (let i = 0; i < 5 && await page.locator('div[aria-hidden="true"][class*="_mask_"]').count(); i += 1) {
  const cont = page.getByRole('button', { name: /Continue|Got it|Close/i })
  if (await cont.count()) await cont.first().click().catch(() => {})
  else await page.keyboard.press('Escape')
  await page.waitForTimeout(1200)
}
```

---

### 23. `noExternal` 已废弃，内联依赖要用 `deps.alwaysBundle`

**现象**：tsdown 报 `` `noExternal` is deprecated. Cannot be used with `deps.alwaysBundle`. ``
**原因**：tsdown 0.22 改用了 `deps.alwaysBundle`；共享 preset 已经设置了它。
**修法**：要在产物里内联某个依赖（让 bundle 自包含），写

```ts
clientBundle(import.meta.dirname, ['lib/types/index.js'], {
  lib: { deps: { alwaysBundle: [/^fast-xml-/] } },
})
```

判断是否内联成功：`grep -E "^import .* from ['\"](lib-name)" lib/index.js` 应该为空，产物体积会明显变大。

---

### 24. `rsync --delete` 会把仓库独有的目录删掉

**现象**：从源码目录同步回独立仓库后，仓库里的 `bundle/`（或其他只存在于仓库的产物目录）整块消失，还被一起提交了。
**原因**：`--delete` 让目标与源完全一致，源里没有的目录就被删除。
**修法**：同步时显式排除：`rsync -a --delete --exclude bundle --exclude .env ...`；提交后核对 `git ls-files <dir> | wc -l` 与磁盘文件数。

---

### 25. 独立仓库装依赖：位置和 lockfile

**现象**：在插件仓库目录里跑 `pnpm install` / `npm install` 失败（`EUNSUPPORTEDPROTOCOL: workspace:*` 或找不到 workspace）。
**原因**：`workspace:*` 只在 pnpm workspace 内解析；插件的真身虽然是独立仓库，但它被符号链接或复制进了 harness 的 `packages/*/*`。
**修法**：在 **harness 根目录**跑 `pnpm install --filter <包名>`。副作用是根 `pnpm-lock.yaml` 多一条 importer 记录 —— **这条不要提交**，CI 上没有该目录。

---

## 一页纸总结

| 阶段 | 最容易漏 |
|---|---|
| 建包 | `tsconfig.json`、聚合登记、路径别名 |
| 写代码 | 入口 `.ts` 不能是 `.tsx`、文案不能硬编码、不能值导入 |
| 元数据 | version / license / description / 八语言 locale / README 四章节 |
| 加载 | 实验包只能 overlay，不能进正式 bundle |
| 验收 | 全量构建过了 ≠ 包内 bundle 能过；门禁全绿 ≠ 界面里看得到 |
