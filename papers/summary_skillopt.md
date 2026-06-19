# SkillOpt: Executive Strategy for Self-Evolving Agent Skills

**Paper:** arxiv 2605.23904
**URL:** https://arxiv.org/abs/2605.23904
**Authors:** Yifan Yang, Ziyang Gong et al. | Microsoft Research & Shanghai Jiao Tong & Tongji & Fudan

---

## 论文目标

将 **agent skill**（一段自然语言文本，包含程序性知识、工具策略、输出格式、失败模式等）作为 **外部可训练状态**，用类似深度学习的优化 Discipline 来训练它——而不是手动编写、一次生成或松散的自修正。

核心问题：**如果 skill 是 adaptation layer，应该如何优化它？**

---

## 核心思想

**将 skill document 当作外部权重来训练**，用 optimizer model 将 scored rollouts 转化为 bounded add/delete/replace edits，只有在 held-out validation 上严格提升才接受。

训练过程：
- **目标模型**（frozen）：执行任务，产生 rollout trajectories
- **Optimizer 模型**：分析成功/失败轨迹，提出结构化编辑
- **Validation Gate**：只在 held-out selection split 上提升才接受
- **部署产物**：`best_skill.md`（300~2000 tokens）

---

## 方法：训练循环

### Forward Pass：Rollout Evidence
- 目标模型在 $D_{train}$ 上用当前 skill 执行 rollout batch
- 收集轨迹：任务元数据、消息、工具调用、观测、最终答案、verifier feedback 等
- Evidence 单元：small batch 更新快但噪声大；large batch 暴露更多重复模式

### Backward Pass：Minibatch Reflection
- 将成功/失败分开，分组为 reflection minibatches
- **失败 minibatch**：propose missing/corrective rules
- **成功 minibatch**：preserve already-working behaviors
- 每组返回结构化 add/delete/replace edits
- **分层合并**：先分别合并失败/成功驱动，再以失败优先合并

### Bounded Text Updates（文本学习率）
- **Edit budget $L_t$**：每步最多应用的编辑数量——这是文本空间的学习率
- 防止无界 rewrite 抹掉有用规则、引入不兼容指令或过拟合
- 支持 constant、linear、cosine、autonomous 调度
- 两种模式：**patch mode**（局部操作）/ **rewrite mode**（全文重写）
- Slow-update 字段受保护，step-level edits 无法覆盖

### Validation Gate + Rejected-Edit Buffer
- 候选 skill 在 $D_{sel}$ 上评估
- **严格 gate**：只有严格 > 当前 selection 分数才接受（平局拒绝）
- 被拒绝的编辑存入 epoch-local buffer，包含失败模式和被拒绝的编辑
- 后续 reflection 调用收到此 buffer，避免重复失败编辑

### Epoch-wise Slow/Meta Update
- **Slow Update**：epoch 结束时，比较相邻 epoch 的 skill，找出 regressions、persistent failures、stable successes
- optimizer model 写入受保护 slow-update 字段，仍通过 validation gate
- **Meta Skill**：optimizer-side only，总结哪些编辑模式有帮助/被拒绝/持续失败
- 预置到未来 optimizer prompts 中，但**不随 skill 部署**
- 分离关注点：部署的 skill 保持轻量，训练侧受益于更丰富的编辑过程记录

---

## 关键设计原则

1. **目标模型固定**：只训练文本 skill，不改模型权重
2. **Held-out validation gate**：防止未验证的反思积累
3. **Minibatch 合并**：最终编辑代表重复证据而非单个样本
4. **Edit budget = 文本学习率**：允许早期大步修改、后期小步精调
5. **部署 skill 保持轻量**：可读、可审计、可手动编辑

---

## 实验结果

### 主结果：52/52 cells 最佳或并列最佳

**Direct Chat (GPT-5.5)：**

| Benchmark | No Skill | SkillOpt | Gain |
|---|---|---|---|
| SearchQA | 77.7 | 87.3 | +9.6 |
| SpreadsheetBench | 41.8 | 80.7 | +38.9 |
| OfficeQA | 33.1 | 72.1 | +39.0 |
| DocVQA | 78.8 | 91.2 | +12.4 |
| LiveMath | 37.6 | 66.9 | +29.3 |
| ALFWorld | 83.6 | 95.5 | +11.9 |
| **Average** | **58.8** | **82.3** | **+23.5** |

**Codex Harness (GPT-5.5)：** +24.8 avg gain over no skill
**Claude Code Harness (GPT-5.5)：** +19.1 avg gain over no skill

比每个 per-cell 竞争者（human/one-shot LLM/Trace2Skill/TextGrad/GEPA/EvoSkill）平均高出 **+5.4 分**。

### 跨模型泛化
- GPT-5.4 上优化的 skill 迁移到 GPT-5.4-mini/nano 均有正收益
- 1 个 case 中迁移性能**超过** in-domain 优化（LiveMath GPT-5.4-nano）

### 跨 Harness 泛化
- Codex 训练的 Spreadsheet skill 迁移到 Claude Code：**+59.7 绝对提升**（22.1→81.8）
- Claude Code → Codex：**+43.6 提升**

### 跨 Benchmark 泛化
- OlympiadBench 训练的 skill 迁移到 Omni-MATH：三个模型尺度均正收益

### Learned Skill 特点
- **Compact**：最终 skill 仅 379~1995 tokens（中位数 ~920）
- **Edit economy**：仅需 1~4 次 accepted edits 即达到大收益
  - LiveMathematicianBench 的 +29.3 来自**单次** accepted edit
- **Procedural**：学习到的是通用程序性规则，而非实例特定知识

---

## Ablation 关键发现

| 组件 | 移除效果 |
|---|---|
| 文本学习率（bounded update） | SpreadsheetBench: 77.5→75.7 |
| Rejected buffer | SpreadsheetBench: 77.5→72.9 |
| Meta skill + Slow update | **SpreadsheetBench: 77.5→55.0（最大跌幅 -22.5）** |
| Batch size（8→full epoch） | 波动仅 ±2pt，不敏感 |
| Learning rate scheduler | constant/cosine/linear 差异小 |

---

## 与 GEPA、MLEvolve 的比较

| 维度 | GEPA | MLEvolve | SkillOpt |
|---|---|---|---|
| 优化对象 | 模块 prompt | 搜索策略 + 记忆 | **Skill document（外部文本状态）** |
| 候选选择 | Pareto frontier | Progressive MCGS + graph | Validation gate |
| 反馈来源 | 文本 trace + 评估反馈 | 执行反馈 + memory | **Rollout trajectories + minibatch reflection** |
| 更新方式 | 反射式 prompt 变异 | 图搜索 + 演化 | **Bounded add/delete/replace edits + slow/meta update** |
| 关键创新 | Pareto selection | Graph reference edges | **Validation gate + rejected buffer + epoch slow update** |
| 部署形式 | 优化后的 prompt | 搜索算法 | **best_skill.md（可迁移文本产物）** |

---

## 与 scenegen_agent 项目的联系

SkillOpt 的核心思想——**将自然语言 artifact 当作外部可训练状态**——与 scenegen 高度契合：

1. **Validation Gate 机制**：
   - 每次规划迭代后，在 held-out scenario 子集上验证 plan 质量
   - 只有在 validation 上提升的 plan 才被接受，避免有害编辑积累
   - 对应 skill 中 rejected buffer → 避免重复同类的 collision 错误

2. **Bounded Edit Budget（文本学习率）**：
   - 每次规划修正限制在 1~4 个改动（类似 $L_t$），保持规划稳定性
   - Cosine 调度 → 早期大步探索、后期小步精调
   - 可迁移到 planning prompt 的迭代修改

3. **Slow/Meta Update（长时域记忆）**：
   - 跨 epoch 保留稳定的规划方向，防止短期波动破坏长期策略
   - 对应场景级别的 meta-planning guidance

4. **Skill artifact 的可迁移性**：
   - 一个场景上优化的 planning skill 可以迁移到相似场景
   - 跨 harness 的正迁移（Codex→Claude Code）启示：scenegen 的 collision-handling skill 可跨环境复用

5. **Minibatch Reflection + Hierarchical Merge**：
   - 成功/失败分别分析 → 分层合并 → 优先级排序 → bounded 应用
   - 对应 planning agent 的多场景经验聚合机制

---

## 详细算法流程

### 完整伪代码

```python
def SkillOpt(target_model, optimizer_model, harness, D_train, D_sel, D_test,
             s_0, epochs=4, L_t=4, batch_size=40, minibatch_size=8):
    s_cur = s_best = s_0
    score_cur = score_best = evaluate(target_model, harness, s_0, D_sel)
    cache = {hash(s_0): score_cur}
    buffer = []
    meta = ∅

    for epoch in range(1, epochs + 1):
        batches = shuffle(D_train).split(batch_size)
        buffer = []  # epoch-local buffer

        for step_batches in accumulate(batches, factor=1):
            # === Forward Pass ===
            trajectories = []
            for x in step_batches:
                tau = execute(harness, target_model, x, s_cur)
                trajectories.append(tau)

            # === Backward Pass: Minibatch Reflection ===
            failures, successes = split_by_outcome(trajectories)
            fail_mbs = partition(failures, minibatch_size)
            succ_mbs = partition(successes, minibatch_size)

            # Parallel minibatch analysis
            fail_patches = [analyze_failure(mb) for mb in fail_mbs]
            succ_patches = [analyze_success(mb) for mb in succ_mbs]

            # Hierarchical merge
            merged_fail = merge_failure(fail_patches, buffer)  # buffer = rejected edits
            merged_succ = merge_success(succ_patches)
            final_merged = final_merge(merged_fail, merged_succ)  # failure priority

            # Ranking + clip to learning rate
            ranked = rank(final_merged,
                          criteria=['systematic_impact', 'complementarity',
                                    'generality', 'actionability'])
            top_edits = ranked[:L_t]

            # === Bounded Update ===
            s_cand = apply_edits(s_cur, top_edits)

            # === Validation Gate ===
            h = hash(s_cand)
            if h in cache:
                score_cand = cache[h]
            else:
                score_cand = evaluate(target_model, harness, s_cand, D_sel)
                cache[h] = score_cand

            if score_cand > score_cur:
                s_cur = s_cand
                score_cur = score_cand
                if score_cand > score_best:
                    s_best = s_cand
                    score_best = score_cand
            else:
                buffer.add({'edits': top_edits,
                            'failures': observed_failures,
                            'drop': score_cur - score_cand})

        # === Epoch-wise Slow/Meta Update ===
        if epoch >= 2:
            slow_content = cross_epoch_analysis(s_prev, s_cur, buffer, D_sel)
            s_cand_slow = inject(s_cur, slow_content)
            if evaluate(target_model, harness, s_cand_slow, D_sel) > score_cur:
                s_cur = s_cand_slow
            meta = update_meta(meta, buffer, s_prev, s_cur)
            s_prev = s_cur

    # === Final Evaluation ===
    score_test = evaluate(target_model, harness, s_best, D_test)
    return s_best, score_test
```

### 分层合并详细流程

```
Step 1: 分离
  trajectories → [failures] + [successes]

Step 2: 分组为 minibatches（每组 minibatch_size=8）
  failures → [F_mb1, F_mb2, ..., F_mbm]
  successes → [S_mb1, S_mb2, ..., S_mbm]

Step 3: 独立并行分析（每组 → 1 个 patch proposal）
  F_mb → LLM("analyze_failure") → patch (max L edits)
  S_mb → LLM("analyze_success") → patch (max L edits)

Step 4: 分组合并
  merge_failure([F_patches], buffer=rejected_edits)
    → deduplicate + resolve conflicts + prevalent-pattern bias
  merge_success([S_patches])
    → deduplicate + keep only patterns NOT already in skill

Step 5: 失败优先合并（final merge）
  final_merge(merged_fail, merged_succ)
    → failure patches take priority
    → success patches only supplement uncovered patterns

Step 6: 排序裁剪
  rank(final_merged) by [impact × complementarity × generality × actionability]
  clip to L_t
```

---

## 关键超参数一览

| 超参数 | 默认值 | 说明 |
|---|---|---|
| Epochs | 4 | 完整遍历 D_train 的轮次 |
| Rollout batch size | 40 | 每步从 D_train 采样的任务数 |
| Reflection minibatch | 8 | 每个 minibatch 的轨迹数 |
| Textual learning rate L_t | 4 | 每步最多 edits 数 |
| LR schedule | cosine | decay to floor=2 |
| Slow update samples | 20 | 跨 epoch 比较的采样数 |
| Gate criterion | strictly greater | 平局拒绝 |
| Split ratio (train:sel:test) | 2:1:7 | 默认数据切分比例 |

---

## 与 scenegen_agent 的详细对应

| SkillOpt 组件 | scenegen_agent 中的对应 |
|---|---|
| **Forward Pass** | 用当前 planning strategy 执行 scenario batch → 收集 collision events + 执行结果 |
| **Minibatch Reflection** | collision cases 分组：失败类（为什么发生 collision）vs 成功类（哪些 planning 策略有效） |
| **Bounded Edit Budget** | 每次 strategy 修正 ≤ N 个规则，避免一次性重写整个 strategy document |
| **Validation Gate** | 在 held-out scenario 子集上验证修改，只有 validation 提升才保留 |
| **Rejected Buffer** | 记录"哪种 collision 处理方式之前失败了"，避免重复相同的错误修复 |
| **Slow/Meta Update** | 跨多轮迭代识别"长期有效的规划模式"，保护字段防止短期调整破坏全局一致性 |
| **best_skill.md** | best_planning_strategy.md，部署时直接注入 planning agent context |

---

## 关键公式

- **搜索目标**：$s^*_{sel} = \arg\max_{s \in \mathcal{C}(D_{tr})} \frac{1}{|D_{sel}|}\sum_{x \in D_{sel}} r(s)$
- **Validation gate**：$s_{new}$ 被接受当且仅当 $score(s_{new}, D_{sel}) > score(s_{cur}, D_{sel})$（严格大于）
- **Textual learning rate**：edit budget $L_t$ 限制每步最多 $L_t$ 个编辑

---

*Summary generated from arxiv 2605.23904 LaTeX source using read-arxiv-paper skill.*
