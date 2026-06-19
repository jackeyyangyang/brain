# GEPA: Genetic-Pareto Reflective Prompt Evolution

**Paper:** arxiv 2507.19457 | ICLR 2026
**URL:** https://arxiv.org/abs/2507.19457

---

## 论文目标

优化 **Compound AI System**（复合 AI 系统，由多个 LLM 模块组成）的 prompt，核心挑战是 **sample efficiency**——RL 方法（如 GRPO）需要数万条 rollouts 才能有效，而 rollouts 在实际场景中代价高昂。

---

## 核心创新：GEPA 三要素

### 1. Reflective Prompt Mutation（反思式 prompt 变异）

**直觉**：LLM 执行产生的自然语言 trace（执行轨迹）本身就包含了丰富的诊断信息。配合评估函数的文本反馈（compiler error、rubric 评分等），LLM 可以做隐式归因——判断哪个模块的 prompt 需要改进，并生成新 prompt。

**流程**：
1. 选择一个候选（candidate）和一个目标模块（round-robin 轮换）
2. 在 minibatch 上跑 rollout，收集 trace + 评估反馈
3. 用 meta-prompt（见下）让 LLM **反思**：归因成功/失败到 prompt 元素，提出新指令
4. 新 prompt 在 minibatch 上验证分数提升 → 加入候选池

**Meta-prompt 原文**：
```
I provided an assistant with the following instructions to perform a task for me:
<current instruction>
The following are examples of different task inputs provided to the assistant
along with the assistant's response for each of them, and some feedback on how
the assistant's response could be better:
<Inputs, Outputs and Feedback for minibatch of examples>
Your task is to write a new instruction for the assistant.
Read the inputs carefully and identify the input format and infer detailed
task description. Read all assistant responses and feedback. Identify niche and
domain-specific factual information about the task and include it in the
instruction. The assistant may have utilized a generalizable strategy—if so,
include that in the instruction as well.
Provide the new instructions within ``` blocks.
```

**关键洞察**：单次变异可以产生大幅、有效的行为更新（对应数万条 rollouts 的 RL）。

### 2. Pareto-based Candidate Selection（Pareto 候选选择）

**问题**：朴素策略（永远选最优候选）容易陷入局部最优——一旦找到主导策略就停止探索。

**GEPA 方案**：
- 对每个训练实例 $i$，记录所有候选中的最优分数 $s^*[i]$
- 筛选出在至少一个实例上达到最优的候选（"winning"策略集合）
- 剔除被其他候选严格支配的候选
- 按"在多少个实例上最优"作为概率权重，随机采样候选

**效果**：平衡探索与利用，避免局部最优，保留多样化策略。

### 3. Genetic Evolution Loop（遗传进化循环）

候选池 $\mathcal{P}$ 从基线开始，迭代产生新候选：
- **Reflective Mutation**：基于文本反馈变异单个模块的 prompt
- **Merge**（可选）：跨候选池 crossover，从不同进化谱系中取各模块最优版本
- 只有 minibatch 上分数提升才加入候选池，并全量评估 Pareto 集

---

## 主要实验结果

| 结论 | 详情 |
|---|---|
| **样本效率** | 比 GRPO（24k rollouts）高 35×，Qwen3 8B 上最高 +20% |
| **超越 MIPROv2** | 在所有 benchmark 和模型上均胜出，合计 +13% vs +5.6% |
| **跨模型泛化** | 在 Qwen3 8B 上优化的 prompt，迁移到 GPT-4.1 Mini 仍有 +9% 提升 |
| **Prompt 更短** | 比 MIPROv2 短 9.2×，推理成本更低 |
| **Pareto > Best** | Pareto 选择比 SelectBest 好 6-8% |

**Benchmark 覆盖**：AIME-2025、LiveBench-Math、HotpotQA、IFBench、HoVer、PUPA

---

## Extended Applications（扩展应用）

1. **Inference-time Search**：作为推理时搜索策略，用于 NPUEval（AMD NPU kernel）和 KernelBench（CUDA），无需训练数据
2. **Adversarial Prompt Search**：反转奖励信号搜索对抗 prompt（AIME 上从 76% 降到 10%）

---

## 与 scenegen_agent 项目的联系

scenegen_agent 同样处理 **多模块 AI 系统**（planning agent、execution agent 等），涉及：
- 模块间 prompt 的协作与优化
- 运行时反馈的利用（collision check 结果等）
- 有限 rollout 预算下的高效适应

**GEPA 的启发**：
1. **反思式变异**可直接用于优化 planning/execution 各模块的 prompt——用模块的执行 trace + 评估反馈做隐式归因
2. **Pareto 候选选择**可以替代当前 planning agent 中可能存在的"选最优"策略，减少陷入局部最优的风险
3. **Merge（crossover）**可以在不同规划策略候选之间做模块级组合
4. **Meta-prompt 框架**非常轻量，适合直接迁移到 scenegen 的 prompt tuning 流程

---

## 算法伪代码（核心）

```
输入: 系统 Φ, 数据集 D_train, 评估 μ, 反馈函数 μ_f, 预算 B, minibatch 大小 b
将 D_train 分为 D_feedback 和 D_pareto
初始化候选池 P = [Φ]

while 预算未耗尽:
    k = SelectCandidate(P)        # Pareto-based
    j = SelectModule(Φ_k)         # round-robin
    M = 从 D_feedback 采样 minibatch
    收集 Φ_k 在 M 上的 trace 和 feedback
    π'_j = UpdatePrompt(π_j, feedback, trace[j])   # LLM reflection
    Φ' = Φ_k 但模块 j 的 prompt 更新为 π'_j
    如果 minibatch 分数提升:
        将 Φ' 加入 P

Return: D_pareto 上平均分最优的候选
```

---

## 关键设计细节

- **Feedback 函数 μ_f**：将评估过程产生的文本 trace（而非仅 scalar reward）返回给优化器，提供高带宽诊断信号
- **minibatch 验证**：新候选先在小 batch 上验证，改善才全量评估，节省预算
- **round-robin 模块选择**：确保每个模块都有机会被优化
- **权重固定**：GEPA 只优化 prompt（Π），不更新 LLM 权重（Θ）

---

*Summary generated from arxiv 2507.19457 LaTeX source using read-arxiv-paper skill.*
