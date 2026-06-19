---
name: runtime-usage-keeper
description: 维护并复用项目最简核心运行方法、关键命令与必填参数。Use when 用户询问代码怎么运行、如何启动 benchmark、需要哪些参数、或希望把最新运行方法沉淀成固定文档时。
---

当用户问“这个项目怎么跑”“最简启动命令是什么”“benchmark 怎么执行”“要配哪些参数”“当前推荐运行方式是什么”时，启用此 skill。

## 核心目标

始终优先复用固定文档：`docs/runtime-core-usage.md`。

这个 skill 做两件事：

1. **回答当前运行方法**：直接从固定文档给出最简命令、必要参数、推荐入口
2. **维护文档新鲜度**：如果本次查看代码后发现入口、参数、默认值、推荐命令有变化，就同步更新 `docs/runtime-core-usage.md`

## 默认工作流

### 场景 1：用户只是在问怎么运行

步骤：

1. 先读 `docs/runtime-core-usage.md`
2. 直接回答：
   - 最简启动命令
   - 必要环境变量
   - 常用命令
   - 关键参数
3. 只有当文档明显不够或用户追问时，才再去读 README / CLI 源码

### 场景 2：用户问某条链路怎么跑

例如：

- benchmark 怎么跑
- pairwise 怎么跑
- session html 怎么生成
- render gif 怎么跑

步骤：

1. 先读 `docs/runtime-core-usage.md`
2. 若已有对应章节，直接回答
3. 若没有或怀疑过期，再检查：
   - `README.md`
   - `src/scenegen/cli.py`
   - 相关 pipeline / service CLI 文件
4. 如发现使用方式已变化，更新 `docs/runtime-core-usage.md`

### 场景 3：用户要求“顺手记下来”

如果这次对运行方式做了确认、修正、补充：

1. 更新 `docs/runtime-core-usage.md`
2. 保持内容精简，只保留：
   - 推荐入口
   - 最小可运行命令
   - 必要参数
   - 常见变体
   - 输出目录 / 结果位置
3. 不写大段背景，不复制整份 README

## 文档维护原则

更新 `docs/runtime-core-usage.md` 时必须遵守：

- **最简优先**：只保留最常用、最核心的运行方式
- **参数导向**：重点写清楚哪些参数必须设、哪些可选
- **避免过期**：不确定的命令不要写进去
- **不抄大文档**：不要把 README 大段搬运过来
- **有变更才更新**：没有变化就不要重写

## 优先检查的文件

当需要核实最新运行方式时，优先看：

1. `docs/runtime-core-usage.md`
2. `README.md`
3. `src/scenegen/cli.py`
4. `src/scenegen/benchmarks/pipelines/gen_render.py`
5. `src/scenegen/benchmarks/pipelines/eval_run.py`
6. `src/scenegen/benchmarks/pipelines/eval_pairwise.py`
7. `src/scenegen/session/html_viewer.py`
8. `src/scenegen/session/render_gif.py`

## 输出要求

回答用户时，优先按下面顺序输出：

1. **最简命令**
2. **必填参数 / 环境变量**
3. **常见变体**
4. **产物位置**
5. **如果我已更新文档，明确说明已同步到 `docs/runtime-core-usage.md`**

## 成功标准

一个好的回答应满足：

- 先给可直接运行的命令
- 明确哪些参数必须设
- 不让用户再去翻大量源码
- 如果运行方法变了，文档已同步更新
