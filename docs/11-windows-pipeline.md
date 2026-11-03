# 11 Windows 侧流水线：让 Codex 当 worker 的实测记录（0001–0004）

这一篇和 01–10 不是同一条线：

- **01–10** 记的是 **Linux/WSL 侧**怎么把一个插件包真正写出来、过大半门禁。
- **这一篇**记的是同一件事在 **Windows 侧**上走了另一条路：不自己写代码，而是把活派给 Codex，
  DSH 只做工头。里面有四轮实测的数据、四类边界的表现，以及**一份失败的任务书**。

写它的理由很简单：换一台机器、换一个会话接手时，最容易犯的错就是拿一边的经验去套另一边。

---

## 0. 一句话结论

> 派活给 Codex 的成本，**九成在任务书，不在模型**；而跨进程/跨沙盒的失败，
> 表现永远像"代码写得不对"，实际都是**权限边界换了一张脸**。

---

## 1. 四轮实测（Windows，2026-11-03）

| 轮次 | 做法 | 结果 |
|---|---|---|
| 0001 | 手装 Codex CLI 0.166.0，`codex exec` | 代码正确；但 Codex 在沙盒里跑了 `git init`，`.git` 归沙盒账户，git 报 `dubious ownership` |
| 0002 | DSH 官方 `subagent_codex` 插件 | 默认 `permissionMode: never` → Codex 回落 `read-only`，**一个文件都写不出**；改 `approve-for-me` 后才通 |
| 0003 | 0.153.4 与 0.166.0 的 A/B | **两版行为完全一致**：同沙盒后端、同所有者、同 git 结果 |
| 0004 | 独立 `CODEX_HOME` 隔离个人配置 | 隔离有效、热加载生效；但单回合上下文只降 6%，且**丢掉了 model 配置** |

## 2. 关键事实（可直接引用）

- DSH 插件 `@deepseek-ai/dsh-subagent-codex` 绑的是**包内自带的 Codex 0.153.4**，
  和系统装的 CLI 无关；换 CLI 不影响 DSH。
- **0.153.4 不支持 MXC**（feature 列表里没有 mxc 项）；0.166.0 有 `prefer_mxc`，但默认关闭。
- Windows 沙盒后端是 `[windows] sandbox = "elevated"`，落到本地账户
  `AAEVIRU\CodexSandboxOffline`：**所有由 Codex 写出的文件所有者都是它**。
- **只要 `.git` 由宿主创建**，文件所有权污染不影响 git（ACL 全是继承项，宿主有 Modify）。
  真正炸掉的是"Codex 自己 `git init`"。
- `--ignore-user-config` 会连 `[windows] sandbox = "elevated"` 一起忽略，
  **`workspace-write` 直接退化成只读**——这个参数在 Windows 上等于关掉写能力。
- Codex App Server 每回合都发 `thread/tokenUsage/updated`（含 cached input、reasoning、上下文窗口），
  但 **DSH Provider 不采集**（源码里 `tokenUsage` 出现 0 次），父会话拿不到真实用量。

## 3. 成本数据（同一台机器，同一账号）

| 场景 | input tokens | 说明 |
|---|---|---|
| 手写 CLI + 全量配置（0001） | 492,027 | 加载了 MCP（cua/node_repl）、插件、skills |
| `-c 'mcp_servers={}'` | 32,594 | 同一任务，**降 15 倍** |
| app-server 单回合探针 | ~14,600 | 只回两个字 |
| 独立 `CODEX_HOME` vs 默认 | 13,784 vs 14,653 | 只降 ~6% |

结论：上下文成本的大头是 **MCP 与插件注入的工具面**，不是"个人配置"，更不是 Codex 版本。

## 4. 同一类边界的三张脸

三天里撞到的三个"看起来不同的故障"，其实是同一件事：

| 现象 | 真实原因 |
|---|---|
| `git: dubious ownership` | 沙盒账户持有 `.git`（Windows 上 Codex 用本地账户跑命令） |
| `writing is blocked by read-only sandbox` | 权限模式没传下去，Codex 回落到原生默认 `read-only` |
| 受限子进程 HTTPS 全断（`Schannel: SEC_E_NO_CREDENTIALS`） | DSH 沙箱令牌拿不到 Windows 加密凭证——**不是网络问题** |

Linux 侧对应的是坑 12（`~/.dsh` 只读 → `EROFS`）和坑 17（Codex 写不了文件）。
**换成 Linux 边界没有消失，只是报错方式变了。**

## 5. 一份失败的任务书（供对照）

给 Codex 的第一版任务书里写了：

```
5. src/client/index.tsx —— 导出 inject 和 apply ...
```

两处硬伤：

1. **`index.tsx` 是错的。** `clientBundle` 在包内构建时入口是写死的字面量
   `src/client/index.ts`，不做扩展名回退。更阴的是：**全量构建能过、包内 bundle 会挂**。
   详见 [02](02-client-entry.md)。
2. **写成"需求描述"而不是"任务书"。** 那条 brief 是自然语言约束
   （"别 import 值""样式用 token"），而有效的写法是
   **原始报错 + 原因 + 具体到文件的修法 + 可照抄的参照**。详见 [08](08-codex-workflow.md)。

结论：**任务书的粒度决定成败**。同样一个 Codex，在 docs/08 那种粒度下十个回合没有一次理解错误；
在这份粒度下，第一个包根本构建不出来。

## 6. 下一阶段的起点

- 骨架已经有了：`packages/experimental/hello-page/`（本仓库不复制，见 hello-nodejs 仓库）。
- 真要往上加能力时，**先定登记面**（座位 / 服务 / 工具），再写代码；
  本套门禁对"缺一处登记"的报错极其模糊，表现是"界面里没有它"。
- 交接格式：**运任务书，不运上下文**。工头负责验收，验收必须自己跑命令，不能读 Codex 的总结。
