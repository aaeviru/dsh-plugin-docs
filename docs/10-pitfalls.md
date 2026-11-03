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

## 一页纸总结

| 阶段 | 最容易漏 |
|---|---|
| 建包 | `tsconfig.json`、聚合登记、路径别名 |
| 写代码 | 入口 `.ts` 不能是 `.tsx`、文案不能硬编码、不能值导入 |
| 元数据 | version / license / description / 八语言 locale / README 四章节 |
| 加载 | 实验包只能 overlay，不能进正式 bundle |
| 验收 | 全量构建过了 ≠ 包内 bundle 能过；门禁全绿 ≠ 界面里看得到 |
