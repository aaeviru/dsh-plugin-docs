# 12 装进桌面版：从源码包到 profile 安装

01–10 讲的是**在源码树里开发**（`--patch` + `dev:web`）。这一篇讲的是另一件事：
把已经写好的包**装进正在运行的 DSH Desktop**，让它在真实 profile 里跑起来。

两者不是同一套机制，混用会卡在"文件都对、界面里没有它"。

---

## 0. 一句话结论

> 源码形态和安装形态是**两份不同的清单**：开发用 `cordis.source.patch.yml`，
> 安装用 `dsh.bundle.patch` + `cordis.patch.yml`。而且安装加载的是**构建产物**，不是 `src/`。
> 宿主提供的包保留为 `peerDependencies`；其余 DSH 包和本例用到的库内联进 `lib/`，避免在 profile 里安装第二套宿主服务。

---

## 1. 先确认三件事，缺一件就别往下走

| 检查 | 怎么查 | 为什么 |
|---|---|---|
| 有没有 `lib/` 产物 | `ls lib/` 看 `index.js`、`client.js` | 包的 `exports` 指向 `lib/*`，profile 不编译 TypeScript |
| 入口构建过不过 | `pnpm --filter <包名> run bundle` 退出码 0 | 全量构建过了 ≠ 包内 bundle 能过（见 [02](02-client-entry.md)） |
| 客户端依赖了什么 | `grep -o 'require("[^"]*")' lib/client.js \| sort -u` | 跨包值导入在别的机器上可能解析不到 |

最后一条查的是客户端模块加载器请求的行：`react` 这类平台内置行须由目标版本提供；其他 `@deepseek-ai/*` 行也必须已注册且版本兼容。声明 npm peer 不能替客户端注册一行。

Host 产物另查外部 `import`：宿主明确提供的包写进 `peerDependencies`，例如本例的 `@deepseek-ai/cordis`；其他依赖内联或声明为可独立安装的叶子库。不要把宿主服务包的 peer 链扩成插件的 `dependencies`。`link:` 安装不会替目标目录安装这些依赖。

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

`package.json` 相对源码包有五处必须改：

```jsonc
{
  "name": "@deepseek-ai/dsh-experimental-hello-page",   // 必须与 lib/client.js 里烘焙的 id 一致
  "type": "module",
  "main": "lib/index.js",
  "exports": { ".": "./lib/index.js", "./client": "./lib/client.js", "./package.json": "./package.json" },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },        // ① 新增：让它成为一个 bundle
    "client": { "platform": "web", "immediately": true } // ② 客户端半边
  },
  "dependencies": {},                                   // ③ 只放可独立安装的叶子库，或留空
  "peerDependencies": { "@deepseek-ai/cordis": "~4.1.0" }, // ④ 宿主包保留为 peer，改成目标宿主支持的真实范围
  "files": ["lib/**/*.js", "lib/**/*.d.ts", "locale/*.json", "cordis.patch.yml"] // ⑤ 包含 patch 和运行产物
}
```

`name` 和 `lib/client.js` 里的 id 必须一致，否则客户端工厂注册到加载器时对不上号：

```js
window.__ModuleLoader__.load({ id: "@deepseek-ai/dsh-experimental-hello-page", factory: … })
```

把源码清单里的 `workspace:*` / `workspace:~` 改成目标宿主支持的真实版本范围，安装清单无需源码开发依赖。不要为了消除 workspace 协议而把宿主 peer 一并删掉。

以 downloader 为例，`dsh-tools`、`dsh-typert-protocol`、`dsh-credentials`、`schemastery`、`zod` 和 `fast-xml-*` 都在源码打包配置中内联，不写进安装包的 `dependencies`。传给 `clientBundle` 的选项如下，完整接法见 downloader 的 [tsdown.config.ts](https://github.com/aaeviru/dsh-downloader/blob/main/tsdown.config.ts)：

```ts
{
  lib: {
    deps: {
      alwaysBundle: [
        /^@deepseek-ai\/dsh-tools$/,
        /^@deepseek-ai\/dsh-typert-protocol$/,
        /^@deepseek-ai\/dsh-credentials$/,
        /^@deepseek-ai\/schemastery$/,
        /^zod$/,
        /^fast-xml-/,
      ],
    },
  },
}
```

打包后检查 `bundle/lib/` 中 Host 和 Typert 各入口的外部导入。downloader 的 Host 入口可以先这样查：

```sh
grep -oE 'from "@deepseek-ai/[^"]+"' bundle/lib/index.js | sort -u
```

应只剩宿主 peer `@deepseek-ai/cordis`。这一条文本检查不覆盖动态导入、其他引号或其他入口，仍须做安装布局与实际 Host 验证。

## 4. 安装

**交付安装使用打包产物（`.tgz`），不要传包目录。**

把包目录作为 `target` 时，profile 的 `package.json` 会记录 `link:` 依赖。`link:` 不为目标包安装依赖；若目标目录没有可解析的运行期依赖，插件导入失败会使 profile 组合挂载不完整。已观察到的桌面报错是 `desktop welcome: Web RPC failed`，其下游原因是 `settings/describe` 拿不到设置服务。

在带有打包脚本的插件仓库中生成 tarball；downloader 使用：

```sh
./scripts/pack.sh  # 产出 dist/<name>-<version>.tgz
```

没有该脚本的包，可在安装 bundle 目录执行 `pnpm pack --pack-destination <输出目录>`。确认打包内容包含 §3 的运行期文件。

```
plugin_manager { action: "install_bundle", target: "F:\\codex-work\\dist\\<name>-<version>.tgz" }
```

或者在 PowerShell 中用 CLI 安装：

```powershell
dsh plugin --profile desktop add 'F:\codex-work\dist\<name>-<version>.tgz'
```

安装器负责写 profile 的依赖与 `dsh.profile.bundles`，并执行包管理安装。**不要手写 profile 的 `package.json` 或 `cordis.patch.yml` 来修补安装。**

装完检查 profile 的 `package.json`：

1. 本插件的依赖为 `file:...tgz`，不能是 `link:`。
2. 在只安装本插件的隔离 profile 中，安装步骤不能额外引入其他 `@deepseek-ai/*` 直接依赖。在已有多个插件的 profile 中，应比较安装前后清单，保留原有插件，检查新增的宿主包。

第 2 条针对重复宿主服务图。把 `dsh-tools` 的 peer 链（`dsh-agent`、`dsh-llm`、`dsh-session`、`dsh-system-prompt` 等）塞进 `dependencies`，已观察到 `deployment:persona-prefix is already registered`，导致新建会话失败。修法是按 §3 保留宿主 peer、内联其余依赖。

插件仓库应带安装布局回归：打包 → 装进隔离 profile → 检查不是 `link:` 且没有新增宿主包 → 在明确提供宿主 peer 的测试环境中导入，输出 `IMPORT_OK`。downloader 的 [scripts/verify-install.sh](https://github.com/aaeviru/dsh-downloader/blob/main/scripts/verify-install.sh) 先检查安装污染，再显式为独立 Node 导入提供 `@deepseek-ai/cordis`；不能把这一步当成真实 Desktop profile 应安装第二份宿主的理由。

结果中至少检查：

- `application: "applied"` → 已生效
- `warnings: []` → 没有版本/权限警告

`packageResult.exitCode` 是包管理器的退出码，`pnpm` 输出里出现 `+ <包名> <版本> <- <路径>`
表示包已安装；退出码、警告和下面的运行期验收仍须一起核对。

## 5. 验证（不要只看"装上了"）

以下六项分别核对安装结构、运行行为和卸载结果：

1. **Host 侧注册**：`cordis_inspect_query` → `Config.listConfigs`，按包名过滤，能看到 entry
   （插件的 `status` 为 `absent` 是正常的——表示它没有声明 Config schema）
2. **客户端座位**：`Slots.listSubTree` 查 `sidebar.panellist` 和 `main`，
   两个 `occupants` 里都要出现你的 id/key
3. **没有碰坏别的**：`Tool.listTools` 看原有工具是否还在
4. **依赖解析**：在实际 Host 中加载并调用插件。独立 Node 导入检查须明确提供声明的宿主 peer，成功后输出 `IMPORT_OK`；裸 profile 中仅因缺少该 peer 而失败，不等于 Desktop 加载失败，也不能直接算验收通过（见 [13](13-link-vs-copy.md)）
5. **宿主没被污染**：比较 profile 安装前后的依赖清单，没有本插件引入的第二套宿主服务包，并确认能新建会话；共享 profile 的原有插件保留
6. **卸载干净**：在隔离 profile 中执行 `dsh plugin --profile <名> remove <包名>`，本插件的依赖、bundle 登记和安装目录被移除；保留其他插件、显式测试 peer 及用户数据

界面上如果没出现，先强刷一次；客户端模块表是启动时取的。

## 6. 装完之后的两个事实

**tarball 安装与源码目录分离。** 包管理器从归档安装文件并按清单解析依赖；它不会把安装包 `link:` 到源码目录。改源码或原来的 `lib/client.js` 不会自动更新已安装副本，必须重新构建、打包和安装。包管理器内部仍可能使用链接，判断安装来源应查 profile 的依赖 specifier。

曾经的目录安装会产生指向 `F:\codex-work\bundle\...` 的链接，只适用于目标目录自带完整依赖的受控开发环境，不能当成交付安装。详见 [13](13-link-vs-copy.md)。

**版本差不必然炸。** 本次插件在 `0.2.2-alpha.3` 的检出里构建，装进 `0.2.1-rc.5` 的桌面版能跑，
因为它只请求了 `react` 这个平台内置行；座位 API（`main` / `sidebar.panellist`）在两边都存在。
这是 hello-page 的历史验收记录，不能推及其他插件或版本。换个插件如果请求了别的行，就要先对齐版本，并遵守安装包的 `engines.dsh` 和 peer 范围；版本兼容最终以目标 Desktop 的实际验收为准。

## 7. 回滚

安装前保存这两个配置文件及现有包管理器 lockfile，并保留上一版 tarball：

```
~/.dsh/profiles/desktop/package.json
~/.dsh/profiles/desktop/cordis.patch.yml
```

卸载走 `plugin_manager { action: "remove_bundle", target: "<包名>" }`；回到旧版则用 `install_bundle` 重装上一版 tarball。若必须从备份恢复，需让配置、lockfile 和包管理安装结果一致；只拷回两个文件或只删插件目录不足以恢复完整安装状态。插件自身的数据另行保留。

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
| 安装被版本要求挡住 | `peerDependencies` 里仍有 `workspace:*` | 换成真实且兼容的版本范围；保留宿主 peer |
| 页面白屏、控制台报模块行缺失 | 客户端请求了目标 profile 里不存在的 `@deepseek-ai/*` 行 | 对齐版本，或改用平台内置依赖 |
| 侧栏有图标但点开空白 | 只注册了 `sidebar.panellist`，没注册 `main` | 两个座位要成套注册 |
| 装完界面没变化、仓库里也没有 `lib/` | `.gitignore` 的 `lib/` 匹配了 `bundle/lib` | 改成 `/lib/`，重新提交并核对 `git ls-files bundle | wc -l` |
| `verify-npm-install-layout` 报版本不符 | `version: "0.0.0"` | 装进 profile 不拦，但进仓库门禁要真实版本号 |
| 桌面启动即报 `desktop welcome: Web RPC failed` | 目录安装记录为 `link:`，目标包缺依赖，导入失败导致组合挂载不完整 | 安装 `.tgz` 并确认 `file:` specifier；继续检查实际导入错误 |
| 新建会话失败：`deployment:persona-prefix is already registered` | 插件把宿主 peer 链带进了 `dependencies` | 保留宿主 peer，其余依赖内联；移除污染后重装并验证会话 |
| 裸 Node 导入缺少 `@deepseek-ai/cordis`，Desktop 却能工作 | 独立 Node 检查没有宿主的 peer 解析上下文 | 显式提供测试 peer 或在实际 Host 验证；不要为检查结果污染真实 profile |
