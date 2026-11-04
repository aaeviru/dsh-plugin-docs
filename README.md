# DSH 插件开发笔记

记录在 **DeepSeek Harness** 里开发一个最小 Web UI 插件（`hello-page`：左侧栏一个图标、点开主面板一个页面）时，真实踩过的坑和必须遵守的规则。

## 为什么会有这个仓库

这个插件看起来只需要五个文件，实际要过十几道门禁。第一次照"最小插件"写出来的版本**既构建不出来、也加载不起来**，而报错信息分散在构建、类型检查、lint 和一堆 `verify-*` 脚本里。

这里把「现象 → 原因 → 修法」记下来，避免下次重新踩。

## 目录

| 文档 | 内容 |
|---|---|
| [01 包骨架](docs/01-skeleton.md) | 一个可构建的插件包到底需要哪些文件 |
| [02 客户端入口](docs/02-client-entry.md) | 为什么入口必须叫 `index.ts`、不能是 `index.tsx` |
| [03 座位注册](docs/03-slots.md) | `main` / `sidebar.panellist` 的注册写法与 inject 边界 |
| [04 纯度门禁](docs/04-purity-gate.md) | 为什么 `import type` 可以、值导入会炸 |
| [05 文案与元数据](docs/05-locale.md) | locale 字典、八语言元数据、README 必备章节 |
| [06 实验包隔离](docs/06-experimental-isolation.md) | 实验包不能进正式 bundle，只能靠 overlay 加载 |
| [07 门禁清单](docs/07-gates.md) | 该跑哪些命令、典型报错长什么样 |
| [08 协作流程](docs/08-codex-workflow.md) | 让 Codex 写代码时的任务书写法 |
| [09 真实验收](docs/09-verification.md) | 怎么证明页面真的出现在界面里 |
| [10 坑位速查](docs/10-pitfalls.md) | 一页纸的踩坑清单 |
| [11 Windows 侧流水线](docs/11-windows-pipeline.md) | 把活派给 Codex 时的沙盒、所有权与成本实测（0001–0004） |
| [12 装进桌面版](docs/12-install-to-desktop.md) | 从源码包到 profile 安装：两份清单、四处必改、三层验证 |

## 一句话结论

> 插件能不能跑，不取决于你写了多少代码，而取决于**三处登记面 + 包骨架 + 门禁元数据**是否齐全。
> 缺任何一处，表现都是"构建不出来"或"界面里没有它"，而不是某个明确的语法错误。

## 环境

本笔记基于：

- Ubuntu 22.04
- Node.js 26
- pnpm workspace（`packages/*/*`）
- DSH 源码检出，插件放在 `packages/experimental/`

> 11 篇另记 Windows 侧的环境（DSH 桌面版 + Codex CLI），两边边界不同，别混用结论。
> 12 篇是最后一步：把包装进 Windows 桌面版的 `desktop` profile，两边都会用到。
