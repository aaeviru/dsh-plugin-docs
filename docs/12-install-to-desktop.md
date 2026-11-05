# 12 装进桌面版：从源码包到 profile 安装

01–10 讲的是**在源码树里开发**（`--patch` + `dev:web`）。这一篇讲的是另一件事：
把已经写好的包**装进正在运行的 DSH Desktop**，让它在真实 profile 里跑起来。

两者不是同一套机制，混用会卡在"文件都对、界面里没有它"。

---

## 0. 一句话结论

> 源码形态和安装形态是**两份不同的清单**：开发用 `cordis.source.patch.yml`，
> 安装用 `dsh.bundle.patch` + `cordis.patch.yml`。而且安装加载的是**构建产物**，不是 `src/`。

---

## 1. 先确认三件事，缺一件就别往下走

| 检查 | 怎么查 | 为什么 |
|---|---|---|
| 有没有 `lib/` 产物 | `ls lib/` 看 `index.js`、`client.js` | 包的 `exports` 指向 `lib/*`，profile 不编译 TypeScript |
| 入口构建过不过 | `pnpm --filter <包名> run bundle` 退出码 0 | 全量构建过了 ≠ 包内 bundle 能过（见 [02](02-client-entry.md)） |
| 客户端依赖了什么 | `grep -o 'require("[^"]*")' lib/client.js \| sort -u` | 跨包值导入在别的机器上可能解析不到 |

最后一条尤其重要：如果 `require()` 里只有 `react` 这类**平台内置行**，跨版本安装通常没事；
一旦出现别的 `@deepseek-ai/*` 行，就要先确认目标 profile 里那行存在且版本兼容。

## 2. 两份清单的区别

源码树里那个文件（开发用）：

```yaml
# packages/experimental/hello-page/cordis.source.patch.yml
- insert:
    - id: experimental-hello-page
      name: './src/index.ts'
```

**`--patch` 启动时**，CLI 用 tsx 直接加载 `./src/index.ts`，不产生 `lib/`、不进 profile。

装进 profile 要的文件（`cordis.patch.yml`，与上者并列存在，不要互相替换）：

```yaml
# bundle/hello-page/cordis.patch.yml
- insert:
    - id: experimental-hello-page
      name: '@deepseek-ai/dsh-experimental-hello-page'
```

差别在 `name`：开发时指向**文件路径**，安装时必须指向**包名**（从 profile 的 `node_modules` 解析）。

## 3. 组装一个可安装的 bundle

从源码包拷一份出来，**只保留运行期需要的东西**（不要 `node_modules`、不要 `src/`、不要 `tsconfig`）：

```
bundle/hello-page/
  package.json
  cordis.patch.yml
  lib/index.js
  lib/client.js
  lib/types/**/*.d.ts
  locale/*.json
```

`package.json` 相对源码包有四处必须改：

```jsonc
{
  "name": "@deepseek-ai/dsh-experimental-hello-page",   // 必须与 lib/client.js 里烘焙的 id 一致
  "main": "lib/index.js",
  "exports": { ".": "./lib/index.js", "./client": "./lib/client.js", "./package.json": "./package.json" },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },        // ① 新增：让它成为一个 bundle
    "client": { "platform": "web", "immediately": true } // ② 客户端半边
  },
  // ③ 删掉 peerDependencies / devDependencies —— 那里是 "workspace:*"，
  //    在 profile 里无法解析，会挡住安装或激活
  // ④ files 里补上 "cordis.patch.yml"
}
```

`name` 和 `lib/client.js` 里的 id 必须一致，否则客户端工厂注册到加载器时对不上号：

```js
window.__ModuleLoader__.load({ id: "@deepseek-ai/dsh-experimental-hello-page", factory: … })
```

## 4. 安装

用 `plugin_manager` 的 `install_bundle`，`target` 传**包目录的绝对路径**：

```
plugin_manager { action: "install_bundle", target: "F:\\codex-work\\bundle\\hello-page" }
```

它会自己做三件事：写 profile 的 `package.json`（`link:` 依赖）、加入 `dsh.profile.bundles`、
执行包管理安装。**不要手写 profile 的 `package.json` 或 `cordis.patch.yml`。**

结果里只看两个字段：

- `application: "applied"` → 已生效
- `warnings: []` → 没有版本/权限警告

`packageResult.exitCode` 是包管理器的退出码，`pnpm` 输出里出现 `+ <包名> <版本> <- <路径>`
就是装上了。

## 5. 验证（不要只看"装上了"）

三层都要查，缺一层就可能出现"装好了但界面里没有它"：

1. **Host 侧注册**：`cordis_inspect_query` → `Config.listConfigs`，按包名过滤，能看到 entry
   （插件的 `status` 为 `absent` 是正常的——表示它没有声明 Config schema）
2. **客户端座位**：`Slots.listSubTree` 查 `sidebar.panellist` 和 `main`，
   两个 `occupants` 里都要出现你的 id/key
3. **没有碰坏别的**：`Tool.listTools` 看原有工具是否还在

界面上如果没出现，先强刷一次；客户端模块表是启动时取的。

## 6. 装完之后的两个事实

**符号链接，不是拷贝。** 安装后 profile 里是：

```
node_modules/@deepseek-ai/dsh-experimental-hello-page → F:\codex-work\bundle\hello-page
```

好处：改 `lib/client.js` 刷新页面就生效，不用重装。坏处：**删掉源目录，插件立刻失效**。

**版本差不必然炸。** 本次插件在 `0.2.2-alpha.3` 的检出里构建，装进 `0.2.1-rc.5` 的桌面版能跑，
因为它只请求了 `react` 这个平台内置行；座位 API（`main` / `sidebar.panellist`）在两边都存在。
换个插件如果请求了别的行，就要先对齐版本。

## 7. 回滚

安装前备份这两个文件，就够回滚：

```
~/.dsh/profiles/desktop/package.json
~/.dsh/profiles/desktop/cordis.patch.yml
```

卸载走 `plugin_manager { action: "remove_bundle", target: "<包名>" }`；
手写恢复则把备份拷回，再删掉 `node_modules` 里的链接。

## 7.5 构建产物到底进没进库

`bundle/` 目录里也有 `lib/`。如果仓库的 `.gitignore` 写的是 **`lib/`（前面没有斜杠）**，这个模式会匹配**任意层级**的 `lib` 目录 —— 于是 `bundle/lib/**` 全部被忽略，`git add -A` 之后你只会看到 `package.json`、`cordis.patch.yml`、`locale/*` 被提交，构建产物一个都没进库。

**提交后必须核对，别信 `git status` 的前几行：**

```powershell
git ls-files bundle | Select-String 'bundle/lib'   # 应该有一串
(git ls-files bundle | Measure-Object).Count       # 应该等于实际文件数
```

```sh
git ls-files bundle | wc -l     # 与 find bundle -type f | wc -l 对比
```

修法：把忽略规则锚定到根目录 —— `/lib/`、`/coverage/`，而不是 `lib/`、`coverage/`。

## 8. 坑位

| 现象 | 原因 | 修法 |
|---|---|---|
| 安装了但界面里没有 | 少了 `dsh.bundle.patch`，或 row 的 `name` 写成了相对路径 | row 的 `name` 必须是包名 |
| `Cannot find package '@deepseek-ai/...'` | 装的目录只有 `src/`，没有 `lib/` | 先在源码树 `pnpm --filter <包名> run bundle` |
| 安装被版本要求挡住 | `peerDependencies` 里的 `workspace:*` | 在 bundle 副本里删掉 peer/dev 依赖 |
| 页面白屏、控制台报模块行缺失 | 客户端请求了目标 profile 里不存在的 `@deepseek-ai/*` 行 | 对齐版本，或改用平台内置依赖 |
| 侧栏有图标但点开空白 | 只注册了 `sidebar.pannellist`，没注册 `main` | 两个座位要成套注册 |
| 装完界面没变化、仓库里也没有 `lib/` | `.gitignore` 的 `lib/` 匹配了 `bundle/lib` | 改成 `/lib/`，重新提交并核对 `git ls-files bundle | wc -l` |
| `verify-npm-install-layout` 报版本不符 | `version: "0.0.0"` | 装进 profile 不拦，但进仓库门禁要真实版本号 |
