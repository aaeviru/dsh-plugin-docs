# 13 本地路径安装：为什么是 link，依赖去哪了

## 现象

本地路径装完，启动时这一行没激活：

```
downloader (@deepseek-ai/dsh-experimental-downloader): failed to import:
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'fast-xml-parser'
imported from F:\...\bundle\lib\index.js
```

包能装上、能进 profile，**但一 import 就崩**。

## 原因（三层，缺一层都解释不了）

**1. 本地目录安装 = pnpm 的 `link:`**

```sh
dsh plugin --profile demo add ./my-plugin      # CLI
```

等价于 `pnpm add link:/abs/path`。profile 的 `package.json` 里会写成：

```json
"@deepseek-ai/dsh-experimental-downloader": "link:F:/codex-work/dsh-downloader/bundle"
```

**2. `link:` 不安装依赖**

它只是把目录挂进 `node_modules`，不会去读目标包的 `dependencies` 再装一遍。
所以包里 import 的第三方库必须**已经躺在包自己的 `node_modules` 里**。

**3. Node 从 realpath 解析，profile 的 `node_modules` 救不了**

`link:` 进来的是个符号链接，Node 默认按真实路径解析（`preserveSymlinks=false`）。
于是 `<你的目录>/lib/index.js` 里的 `import 'fast-xml-parser'` 会从
`<你的目录>/node_modules` 一路往上找——**永远走不到 profile 的 `node_modules`**。

这就是"明明 profile 里装了、还是找不到"的根。

> 注意区别：`@deepseek-ai/*` 那几个平台包不会报错，因为 harness 自己有解析路径；
> 只有真正的 npm 第三方依赖会炸。

## 怎么判断当前装的是 link

```powershell
# profile 的依赖清单
(Get-Content "$env:USERPROFILE\.dsh\profiles\desktop\package.json" -Raw |
  ConvertFrom-Json).dependencies

# node_modules 里是链接还是实体目录
Get-Item "$env:USERPROFILE\.dsh\profiles\desktop\node_modules\@deepseek-ai\<包名>" |
  Select-Object LinkType,Target
```

`LinkType: SymbolicLink` 就是 link。**版本号**（如 `0.2.1-rc.5`）才是真安装。

## 修法 A：开发期，让包自带依赖

官方文档对这种形态的描述是：*"通过链接安装的 checkout 保留自己的 `node_modules`"*。
换句话说，**依赖本来就该装在包目录里**。

```powershell
npm install --prefix <bundle 目录> --no-save --no-package-lock `
  fast-xml-parser@5.11.1 fast-xml-validator@1.4.2
```

- 只写 `<bundle>/node_modules`，不改任何被 git 跟踪的文件
- 代价：**只在这台机器上有效**，别人 clone 你的仓库没有这份 `node_modules`
- 另一个代价：源目录不能删（删了插件立刻失效）

自检：

```powershell
# 产物里还剩哪些外部 import
Select-String -Path .\bundle\lib\index.js -Pattern '^import .* from "' | ForEach-Object { $_.Line }
# 对照 package.json 的 dependencies，缺哪个补哪个
```

## 修法 B：分发期，tarball 或 npm

官方给的两条"用户不需要构建授权"的路：

> - **发布到 npm**，在 `pnpm publish` 时构建好 `lib/`；`dsh plugin add your-package` 安装的就是预构建代码。
> - **交付 tarball**：用 `pnpm pack` 打包；用户执行 `dsh plugin add ./hello-plugin-0.1.0.tgz`。

关于平台：**tgz 不是 Windows 归档格式，是 npm 包格式**。pnpm/npm 解包走 Node 的 tar 实现，
跟操作系统无关——`pnpm add` 从 registry 拉下来的本来就是 tgz。

tarball 安装是**真解包**，不是链接，所以：

- 依赖按 `package.json` 的 `dependencies` 装（前提是你声明了）
- 装完 `LinkType` 不再是 `SymbolicLink`
- 源目录可以删

前置条件：`bundle/package.json` 必须有 `dependencies`，否则 tarball 装完照样缺包。

## 坑位

| 现象 | 原因 | 修法 |
|---|---|---|
| `ERR_MODULE_NOT_FOUND` 指向第三方包 | link 安装 + 包内无 `node_modules` | 修法 A 或 B |
| 补了 `dependencies` 还是找不到 | 只改了清单，没装；或仍是 link | 真的把包装进包内，或改 tarball |
| `git ls-files` 里看不到 `lib/` | `.gitignore` 的 `lib/` 匹配了任意层级 | 改成 `/lib/`，见 [12](12-install-to-desktop.md) |
| 装完 profile 里是 `link:` | 目标传的是目录 | 想要拷贝就用 tarball / npm |
| 删了源码目录插件消失 | link 的本质 | 分发时改用 tarball |

## 一句话

> **link 是开发形态，tarball 是分发型态。**
> 你从目录装，就必然得到链接；想让别人拿到一份能直接跑的副本，就把 `lib/` 和 `dependencies` 一起打包。
