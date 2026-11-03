# 08 协作流程：让 Codex 写插件代码

## 核心结论

> **门禁报错原文，比任何自然语言描述都适合当任务书。**

这次一共派了 10 轮 Codex，前几轮失败和返工几乎都源于任务书粒度，而不是 Codex 的理解能力。一旦指令精确到「文件 + 行 + 期望代码」，执行质量很高，没有一次理解错误。

## 任务书该怎么写

### ✅ 好任务书

```
缺陷 1：包内 bundle 失败。
`pnpm --filter <pkg> run bundle` 报 UNRESOLVED_ENTRY: Cannot resolve entry module src/client/index.ts。
原因是 src/client/index.tsx 有 JSX，而打包预设的入口字面量固定是 src/client/index.ts。
修法：拆成三个文件 —— index.ts（无 JSX）、HelloPage.tsx、HelloIcon.tsx，删除原 index.tsx。
```

特征：

- **现象**：原始报错，一字不改
- **原因**：为什么会这样（给出源码位置）
- **修法**：具体到新建/删除哪些文件
- **参照**：给出可以照抄的现有文件路径

### ❌ 差任务书

```
帮我写一个最小插件，左侧栏一个图标，点开显示页面。
```

问题：仓库有十几道门禁，这个描述对应不到任何一道。

## 给参照文件，不要凭空发明

每一轮都明确写出「照抄哪个文件的哪个部分」：

```
参照：
1. packages/experimental/client-ui-stats/package.json —— manifest 形状
2. packages/client/ui-plugin-manager/src/client/index.ts 第 167-173 行 —— sidebar.panellist 写法
3. apps/web/tests/client-ui-stats-panel.overlay.yml —— 实验包加载方式
```

没有参照时，模型会按"最像的通用写法"生成，然后在门禁上反复翻车。

## 一次只修一类问题

按门禁分组派活，不要一次给十个缺陷——容易改坏已经通过的部分。

本次的派活节奏：

1. 建包（5 个文件）
2. 补构建与加载登记（tsconfig、聚合、overlay）
3. 修构建 + i18n + README + 别名 + 发布策略
4. 补元数据（version/license/description/locale）
5. 修 lint

## 必须写进任务书的硬约束

```
- 不要修改 scripts/ 下的任何文件（不要改门禁）
- 不要改 packages/bundle/ 下任何文件
- 不要 import 任何 @deepseek-ai/* 的值
- 不要运行 git、pnpm install、不要重启 dev server
- 样式只用内联 style + var(--dsw-alias-*)
```

**「不要改门禁脚本」这条尤其重要**：否则模型可能为了让门禁通过而去改判定逻辑。

## 让 Codex 自检，但别信它的自检

任务书里加：

```
如果 shell 可用，跑 `pnpm --filter <pkg> run bundle`，确认退出码为 0；
跑不了就说明原因，不要假装跑过。
```

即使这样，**验收必须自己重跑**。本次多轮里，Codex 的 shell 被沙箱挡住（IPC/网络限制），它如实报告了失败，但也意味着它说的"通过"只能当参考。

## 循环模型

```
写任务书 → 派活 → 自己跑门禁验收 → 把新的报错原文抄进下一份任务书 → 再派
```

每轮验收都应该是**你自己跑命令**，而不是读它的总结。
