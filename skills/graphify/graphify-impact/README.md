# graphify-impact

用于**改动前影响分析、依赖反查、风险评估、改后同步**。

## 核心

- 先 `graphify affected`
- 必要时补 `graphify path`
- 不清楚的节点再 `graphify explain`
- 再定点读源码
- 改完执行 `graphify update .`

## 适合场景

- 重构前评估
- 看谁依赖某个模块
- 判断回归风险
- 决定优先验证哪些流程
