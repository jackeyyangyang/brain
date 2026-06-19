# MLEvolve: Self-Evolving Framework for Automated Machine Learning Algorithm Discovery

**Paper:** arxiv 2606.06473
**URL:** https://arxiv.org/abs/2606.06473
**Authors:** Shangheng Du, Xiangchao Yan et al. | Shanghai AI Lab & East China Normal University

---

## 论文目标

解决 LLM Agent 在长时域 MLE（Machine Learning Engineering）任务中的**自我进化**问题。当前 MLE agents 面临三大阻碍：

1. **跨分支信息隔离**：树搜索只在单条路径内传递信息，无法跨分支共享成功策略
2. **无记忆搜索**：每步规划都孤立做出，无法复用历史经验
3. **缺少层级控制**：规划和代码生成耦合，每次迭代都重写整个方案

---

## 核心三组件

### 1. Progressive MCGS（渐进式蒙特卡洛图搜索）

**将搜索空间建模为有向图**：
- 节点 = 候选解（完整 ML pipeline 代码）
- **Primary Edges** ($E_T$)：父→子生成关系，用于 selection/backpropagation
- **Reference Edges** ($E_{ref}$)：跨分支参考边，连接不同分支中的节点，实现知识流和组合复用（不参与 backprop）

**四类扩展操作**：

| 操作 | 触发条件 | 机制 |
|---|---|---|
| **Primary** | 基线 | 仅从父节点生成，无参考 |
| **Intra-branch Evolution** | 分支内连续失败 | 回顾同分支前 k 个节点，自我反思避免重复错误 |
| **Cross-branch Reference** | 分支停滞 | 从全局 top-N 节点中获取灵感 |
| **Multi-branch Aggregation** | 全局停滞 | 合并多条分支的最优轨迹，创建新分支起点 |

**渐进式探索调度**（熵驱动）：
- 用 UCT 平衡探索/利用，但 exploration constant $c(t)$ 随时间分段递减
- 软切换：概率 $w(t)$ 在 UCT 探索和 Elite-Guided 利用之间切换
- $w(t)$ 逐步降低 → Shannon 熵降低 → 有效分支数从 4.8 降至 2.8（实验验证）

**Multi-Level Stagnation Detection**：
- Branch-level：连续 $\tau_{branch}$ 次无提升 → Intra-branch Evolution → Cross-branch Reference
- Global-level：全局最优连续 $\tau_{global}$ 次无提升 → Multi-branch Aggregation

### 2. Retrospective Memory（回顾记忆）

**静态知识库（冷启动）**：
- 按任务类型组织的候选模型库（CV/NLP/Tabular 等）
- 从开源仓库和比赛平台提炼模型+使用指南
- 初始化时提供 task-relevant priors，降低冷启动错误率

**动态全局记忆（运行时积累）**：
- 每次有效节点执行后，自动积累结构化记录（plan、outcome、analysis、feedback）
- **Hybrid Retrieval**：lexical keyword + FAISS semantic search，通过 RRF（Reciprocal Rank Fusion）融合
- **Stage-aware Retrieval**：
  - Planning 阶段：用 plan 查询，检索成功/失败经验，指导 plan 细化
  - Debugging 阶段：用 error message 查询，检索相似已解决错误

**无需额外 LLM 做显式反思**，经验和知识自动积累复用。

### 3. Hierarchical Planning + Adaptive Code Generation

**Planner-Coder 解耦**：
- Planner：模块级决策，决定 **"改什么"** 和 **"为什么改"**
- Coder：代码级实现，决定 **"怎么改"**，保持现有结构和有效函数

**三种编码模式**（自适应选择）：

| 模式 | 适用场景 | 说明 |
|---|---|---|
| **Base** | 冷启动/无可靠方案 | 从零生成完整代码 |
| **Stepwise** | 复杂多阶段 pipeline | 按 planner 规范逐步生成模块 |
| **Diff** | 已有可行方案 | 对现有代码做精准 diff 编辑 |

### 4. Agent 团队（9 个专业 Agent）

Draft / Improve / Debug / Evolution / Fusion / Aggregation / Code Review / Data Leakage / Result Parse

---

## 实验结果

### MLE-Bench（75 tasks，12h budget）

| 指标 | MLEvolve | 最佳 Baseline |
|---|---|---|
| 平均 Medal Rate | **65.3%** | ~55% |
| Gold Medal Rate | **34.7%** | — |
| Valid Submission | **100%** | — |
| Above Median | **76.0%** | — |

在低/中/高复杂度任务上分别达到 80.3% / 64.0% / 46.7%。

### AlphaEvolve 数学优化（15 tasks）

MLEvolve 在 **11/15** 任务上胜出，超越 AlphaEvolve、AlphaEvolve-v2、OpenEvolve 等专用算法发现方法。

### Ablation

三个组件去掉任意一个都有明显下降：
- 去掉 Progressive MCGS：影响最大（medal rate + beat ratio 双降）
- 去掉 Retrospective Memory：-13.64% medal rate
- 替换自适应代码生成为 one-shot：整体性能下降

---

## 与 scenegen_agent 项目的联系

scenegen_agent 是**场景生成**领域的多模块 AI 系统，面临类似 MLE 的长时域迭代优化挑战：

1. **Progressive MCGS 的图结构** → 可迁移到 scenegen 的规划搜索：
   - 多条 planning 候选路径之间通过 reference edges 共享成功策略
   - 渐进式探索调度避免 planning 陷入局部最优

2. **Retrospective Memory** → scenegen 的经验积累：
   - 每次 collision check / plan 生成/执行的经验自动积累复用
   - stage-aware retrieval 机制（planning 阶段 vs debugging 阶段）可直接对应
   - 无需额外 LLM 做显式反思的设计非常高效

3. **Planner-Coder 解耦 + Diff 模式** → 更稳定的代码迭代：
   - planning 决定"改什么"，执行层决定"怎么改"
   - Diff 模式在已有可行方案时做精准修改，减少重写风险

4. **Intra-branch Evolution** → 对应 scenegen 的分支内历史反思：
   - 连续 collision 后，回顾同类错误的处理方式，避免重复踩坑

---

## 关键公式速查

- **搜索目标**：$s^* = \arg\max_{s \in \mathcal{S}} h(T, s)$
- **UCT 选择**：$\pi_{\text{sel}}(v) = \arg\max_i \left( Q_i + c(t)\sqrt{\frac{\ln(N_v+1)}{N_i+\varepsilon}} \right)$
- **软切换**：$P(\text{UCT}) = w(t)$，其中 $w(t)$ 随时间从 1.0 降至 $w_{\min}$
- **Elite-Guided**：$P(v_i) \propto 1/\text{rank}(v_i)$，从 top-K 全局最优节点中选择
- **RRF 检索融合**：$\text{score}(d) = \alpha \cdot \frac{1}{k+r_{\text{lex}}(d)} + (1-\alpha) \cdot \frac{1}{k+r_{\text{vec}}(d)}$

---

*Summary generated from arxiv 2606.06473 LaTeX source using read-arxiv-paper skill.*
