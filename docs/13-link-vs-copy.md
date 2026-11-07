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

## 装完之后怎么确认"真的装进去了"

逐条跑，全部通过才算换成拷贝安装：

| 判据 | 命令 | 期望 |
|---|---|---|
| 依赖不是链接 | 看 profile 的 `dependencies` | `file:...tgz` 或版本号，**不含 `link:`** |
| 目录是实体 | `Get-Item node_modules\<包名> \| Select LinkType,Target` | 两者都为空 |
| lockfile 有真实解析 | 在 `pnpm-lock.yaml` 里搜包名 | 有 `specifier:` 与完整的 peer 解析链，而不是单个 link importer |
| 能被导入 | 在 profile 目录下 `node -e "import('<包名>').then(()=>console.log('IMPORT_OK'))"` | `IMPORT_OK` |

前三条管的是"安装结构"，第四条管的是"运行时解析"。**它们不是一回事**，下一节就是反例。

## peer-only 包：`IMPORT_OK` 会失败，但插件是好的

把依赖全部内联、只留 `@deepseek-ai/cordis` 作为 peer 之后，会出现这样的组合：

```
装进 profile：plugin_manager → application: applied
功能实测：    download_search("FC2") → Found 75 candidates
导入检查：    IMPORT_FAIL: Cannot find package '@deepseek-ai/cordis'
               imported from .../dsh-experimental-downloader/lib/index.js
```

这不是打包错，是 **peer 的正确行为**：

- pnpm 不会为 `file:` 依赖安装 peer；
- profile 的 `node_modules` 里本来就没有 `@deepseek-ai/cordis`；
- host 加载插件时用的是**自己的** cordis 实例。

如果为了凑 `IMPORT_OK` 把 cordis 也声明成 `dependencies`，profile 里就会多出第二份 cordis，插件拿到的 `Context`/`Service` 与宿主不是同一个类，`instanceof` 会失效——**用错误的修法换一个漂亮的检查结果**。

所以这条判据要写成：

> 在 host 的解析上下文里检查，或允许唯一未解析项就是包声明的那一个 peer。

## 坑位

| 现象 | 原因 | 修法 |
|---|---|---|
| `ERR_MODULE_NOT_FOUND` 指向第三方包 | link 安装 + 包内无 `node_modules` | 修法 A 或 B |
| 补了 `dependencies` 还是找不到 | 只改了清单，没装；或仍是 link | 真的把包装进包内，或改 tarball |
| `git ls-files` 里看不到 `lib/` | `.gitignore` 的 `lib/` 匹配了任意层级 | 改成 `/lib/`，见 [12](12-install-to-desktop.md) |
| 装完 profile 里是 `link:` | 目标传的是目录 | 想要拷贝就用 tarball / npm |
| 删了源码目录插件消失 | link 的本质 | 分发时改用 tarball |
| `IMPORT_OK` 失败但面板正常 | 只剩 peer 未解析，属正确行为 | 放宽判据，不要重复装 peer |
| 换成 tgz 后仍报旧路径的错 | 运行中的 host 还持有上一次加载失败的行 | 重启后再看 |

## 一句话

> **link 是开发形态，tarball 是分发型态。**
> 你从目录装，就必然得到链接；想让别人拿到一份能直接跑的副本，就把 `lib/` 和 `dependencies` 一起打包。
