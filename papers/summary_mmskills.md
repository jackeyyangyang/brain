# MMSkills: Towards Multimodal Skills for General Visual Agents

**Paper:** arxiv 2605.13527
**URL:** https://arxiv.org/abs/2605.13527
**Authors:** Kangning Zhang, Shuai Shao et al. | Shanghai Jiao Tong University & Xiaohongshu & Southeast University

---

## 论文目标

视觉 Agent 的技能（skill）本质上是**多模态的程序性知识**——reuse 不仅取决于"执行什么操作"，还取决于"如何识别相关状态"、"如何解读视觉证据"、"如何决定下一步"。现有 skill 包大多只编码文本 prompt、代码或 learned routines，**忽略了视觉维度的程序性知识**。

核心问题：**视觉 Agent 的 skill package 应该包含什么？如何从公开交互经验中构建？如何让 agent 在推理时不因过多图像上下文而超载、不被参考截图锚定？**

---

## 核心思想

**MMSkills**（Multimodal Skills）：将可重用视觉程序性知识表示为 state-conditioned package，每个 MMSkill = `(D, P, S, K)`：

```
MMSkill = (D, P, S, K)
  D = descriptor        # 紧凑描述符
  P = procedure         # 文本程序（可重用操作模式）
  S = state_cards       # 运行时状态卡集合
  K = keyframes         # 与状态卡对齐的多视角关键帧集合
```

**State Card**（运行时状态卡）是 Agent-facing 状态节点，包含：
- `when_to_use` — 何时使用
- `when_not_to_use` — 何时不使用
- `visible_cues` — 观察什么视觉证据
- `verification_cue` — 如何验证进度/完成
- `available_views` — 可用视图类型

**Keyframe Bundle**（多视角关键帧）：
- `full_frame` — 全局上下文
- `focus_crop` — 局部视觉线索
- `before` — 变化前状态（可选）
- `after` — 目标完成状态（可选）

**关键区分**：MMSkill ≠ "text skill + 截图"，而是**状态条件化程序**，其视觉证据帮助 agent 决定何时 follow、skip 或 verify。

---

## 三大挑战与解法

| 挑战 | 解法 |
|---|---|
| **表征**：多模态 skill package 应该包含什么？ | State-conditioned package = 文本程序 + 状态卡 + 多视角关键帧 |
| **生成**：如何从公开非测试轨迹构建？ | Agentic trajectory-to-skill Generator（五阶段 pipeline） |
| **利用**：如何在推理时不超载且防止被截图锚定？ | Branch Loading（临时分支加载） |

---

## 方法一：Skill Package 表示

### 形式化

```
M = (D, P, S, K)

其中：
  S_j = (when_to_use_j, when_not_to_use_j, 
         visible_cues_j, verification_cue_j, V_j)
  K_j = {K_j^v : v ∈ V_j, v ∈ V}
  V = {full_frame, focus_crop, before, after}
```

### 关键设计

- State Card 是**决策节点**而非图片描述——它规定了何时 follow/skip/verify
- Keyframe 是**参考证据而非坐标模板**——agent 不能复制截图坐标
- Text-only skill 是 MMSkill 的退化形式：`(D, P, ∅, ∅)`

### Branch Loading：推理时利用机制

**问题**：直接插入 skill 到主 context 会造成大量上下文压力，参考截图会与 live observation 竞争，agent 会被视觉相似但不同的截图锚定。

**解法**：Branch Loading = 多模态渐进式披露

```
Stage 1: Gated View Selection
  输入: O_t (live observation), H_{t-1} (history), P_t, S_t
  输出: (J_t, R_t) = selected state cards + selected view types
        V_t = {K_j^v : j ∈ J_t, v ∈ R_{t,j}}

Stage 2: Branch Planning
  输入: O_t, H_{t-1}, P_t, {S_j : j ∈ J_t}, V_t
  输出: G_t = (applicable_t, subgoal_t, plan_t, do_not_do_t, verify_t)

主 Agent: A_t = π_main(O_t, H_t, C_I, G_t)
```

Branch 输出是**结构化决策支持**而非完整 skill package，action grounding 始终绑定到 live observation。

---

## 方法二：Skill Generator

### 五阶段 Pipeline

```
T_d (公开轨迹池)
  ↓ Phase 0: embed + cluster
C_d (语义聚焦簇)
  ↓ Phase 1: cluster-level skill planning
A_d (域规划表)
  ↓ Phase 2: merge
R_d (合并技能规范)
  ↓ Phase 3: text-first drafting
M̂_d (文本草稿)
  ↓ Phase 4: image grounding + audit
M_d (最终多模态技能库)
```

**Phase 0**: 嵌入任务指令 + 轨迹元数据，语义聚类
**Phase 1**: LLM-based agent 为每个簇提出 atomic skills（workflow boundaries、completion conditions、covered task ids）→ 域规划表
**Phase 2**: 去重、合并、泛化，过 broad 的 umbrella skills 被拒绝
**Phase 3**: 不读取图片，生成 descriptor + textual procedure + planned state cards
**Phase 4**: 读取选定的关键帧，构建多视角 bundle，meta-skill-guided auditing

### Meta-Skill 控制

Generator 由可重用的 **multimodal-skill-factory meta-skill** F 控制：
```
G_F: T_d → M_d
```
F 提供可重用的脚本、schema 和质量门控。

视觉 grounding 策略是**保守的**：视图仅在状态识别、转换对比或完成验证时添加，skill 存储诊断状态而非回放演示。

---

## 实验结果

### RQ1: GUI + 游戏任务上的总体性能

**OSWorld（主 benchmark）：**

| 模型 | No Skill | Text-only | MMSkills | Gain |
|---|---|---|---|---|
| Gemini 3.1 Pro | 44.08% | 40.76% | **50.11%** | +6.03 |
| Gemini 3 Flash | 36.65% | 40.27% | **47.97%** | +11.32 |
| Qwen3-VL-235B | 21.34% | 28.57% | **39.17%** | +17.83 |
| Kimi-K2.6 | 34.98% | 39.66% | **46.59%** | +11.61 |
| Qwen3-VL-8B | 10.78% | 14.93% | **25.40%** | +14.62 |

**跨 benchmark 迁移：**
- macOSWorld: Gemini 3 Flash 55.94%→65.73%
- VAB-Minecraft: 所有模型均有提升
- Super Mario Bros: 所有模型均有提升

**关键洞察**：外部多模态程序性知识对**弱模型**价值最大——Qwen3-VL-8B 提升 2.4 倍（10.78%→25.40%）。

### RQ2: Ablation

**Skill Content：**
- State cards 和 multi-view keyframes 均重要
- 移除任一组件均降性能
- 两者互补：state cards 支持状态区分，keyframes 帮助识别对应视觉证据

**Branch Loading：**
- Direct-full loading（直接插入全部内容）损害性能
- View selection 单独效果有限
- Branch loading 已有明显提升，完整两阶段设计最佳

### RQ3: 技能使用与交互动态

| 指标 | 观察 |
|---|---|
| MMSkills 调用频率 > Text-only | Qwen3-VL-235B: 37.50%→65.28% |
| MMSkills 缩短轨迹 | Qwen3-VL-235B: -5.35 步（15.22→9.87） |
| Focus crops 主导 | 79/241/8/24（full/focus/before/after） |

### RQ4: 行为变化

- **减少低层操作负载**：点击份额从 75.8%→63.7%
- **抑制重复轨迹**：精确重复动作从 21.8%→6.2%
- **增强完成意识**：DONE 行为增加

---

## 与 GEPA、MLEvolve、SkillOpt 的比较

| 维度 | GEPA | MLEvolve | SkillOpt | **MMSkills** |
|---|---|---|---|---|
| 优化对象 | 模块 prompt | 搜索策略 + 记忆 | Skill document | **多模态 Skill Package** |
| 核心创新 | Pareto selection | Graph MCGS + memory | Validation gate | **Branch Loading + State Cards** |
| 技能表示 | 文本 | 记忆 + 图 | 文本 + edits | **文本 + 状态卡 + 多视角关键帧** |
| 利用方式 | 直接插入 | 图搜索 | 验证后接受 | **Branch Loading（临时分支）** |
| 反馈来源 | trace + 评估 | 执行 + memory | Rollout + gate | **公开轨迹 + 视觉 grounding** |
| 生成方式 | 人类编写/一次生成 | 演化搜索 | 优化训练 | **五阶段 Generator pipeline** |

---

## 与 scenegen_agent 项目的联系

MMSkills 的核心思想——**多模态状态条件化技能 + Branch Loading**——与 scenegen 的视觉规划高度契合：

1. **State Cards → Scenario Condition Cards**：
   - scenegen 的 scenario 可包含"何时使用此规划策略"的视觉条件
   - 当 agent 进入某个状态时，激活对应的规划 skill
   - 视觉 cue 帮助识别 collision-prone 场景

2. **Multi-view Keyframes → Multi-scenario Evidence**：
   - 关键帧的 `before/after` 对应 collision 前后的状态对比
   - `focus_crop` 对应 collision-prone 操作区域的局部放大
   - 视觉 grounding 比纯文本更精确地定位问题

3. **Branch Loading**：
   - 临时分支机制避免 skill 全部塞入主 context
   - scenegen 可用分支来验证 planning strategy 的适用性
   - 主 agent 收到结构化决策支持而非完整 skill package

4. **五阶段 Generator Pipeline**：
   - 从历史执行轨迹中自动构建可重用 skill
   - 对应 scenegen 从 collision logs 自动生成 collision-handling rules
   - Phase 1→4 的分层生成适合自动构建规划知识库

5. **弱模型价值更大**：
   - Qwen3-VL-8B 提升 2.4 倍的洞察 → 小规模 LLM 可从 MMSkills 中获益更多
   - 对应 scenegen 中 lightweight planning 可显著弥补小模型能力差距

---

## 关键公式

- **Skill Package**: $M = (D, P, S, K)$
- **State Card**: $S_j = (\text{when\_to\_use}_j, \text{when\_not\_to\_use}_j, \text{visible\_cues}_j, \text{verification\_cue}_j, \mathcal{V}_j)$
- **Keyframe Bundle**: $K_j = \{K_j^v : v \in \mathcal{V}_j\}$
- **Branch Output**: $G_t = (\text{applicable}_t, \text{subgoal}_t, \text{plan}_t, \text{do\_not\_do}_t, \text{verify}_t)$
- **Generator**: $\mathcal{G}_\mathcal{F} : \mathcal{T}_d \mapsto \mathcal{M}_d$

---

## 详细算法流程

### Skill Package 形式化

```python
@dataclass
class MMSkill:
    descriptor: str           # D: 紧凑描述符
    procedure: str            # P: 可重用文本程序
    state_cards: List[StateCard]   # S: 运行时状态卡集合
    keyframes: List[KeyframeBundle]  # K: 多视角关键帧集合

@dataclass
class StateCard:
    when_to_use: str         # 何时使用
    when_not_to_use: str     # 何时不使用
    visible_cues: str        # 观察什么视觉证据
    verification_cue: str    # 如何验证进度/完成
    available_views: Set[ViewType]  # 可用视图类型

@dataclass
class KeyframeBundle:
    state_id: int
    views: Dict[ViewType, Image]  # full_frame / focus_crop / before / after
```

### Branch Loading 运行时循环

```python
def branch_loaded_agent(skill_library: List[MMSkill], instruction: str, env):
    H = []  # 主轨迹历史
    C_I = pre_recall(instruction, skill_library)  # 任务级候选集
    
    for t in range(max_steps):
        O_t = env.observe()  # 当前视觉观测
        
        decision = main_agent.decide(O_t, H, C_I)
        # decision = Action(...) 或 SkillRequest(skill_name)
        
        if isinstance(decision, Action):
            env.execute(decision.action)
            H.append((O_t, decision))
        else:
            # Branch Loading
            M_t = get_skill(decision.skill_name)
            
            # Stage 1: Gated View Selection
            J_t, R_t = select_views(O_t, H[-1], M_t.procedure, M_t.state_cards)
            V_t = {M_t.keyframes[j][v] for j in J_t for v in R_t[j]}
            
            # Stage 2: Branch Planning
            G_t = plan_branch(O_t, H[-1], M_t.procedure, 
                             [M_t.state_cards[j] for j in J_t], V_t)
            
            # Main agent 收到结构化指导，仍从 live observation 做 action
            A_t = main_agent.grounded_action(O_t, H[-1], G_t)
            env.execute(A_t)
            H.append((O_t, A_t, G_t))
        
        if env.is_complete():
            return success
    
    return failure
```

### 五阶段 Generator Pipeline

```python
def generate_skill_library(trajectory_pool: List[Trajectory], meta_skill: MetaSkill):
    # Phase 0: Task embedding + clustering
    clusters = embed_and_cluster(trajectory_pool)
    
    # Phase 1: Cluster-level skill planning
    for cluster in clusters:
        proposed_skills = llm_planning_agent(cluster, meta_skill)
        domain_table.extend(proposed_skills)
    
    # Phase 2: Skill merging
    merged = merge_skills(domain_table)  # 去重 + 泛化
    
    # Phase 3: Text-first drafting (no images)
    drafts = []
    for spec in merged:
        draft = draft_skill_text(spec)  # D, P, Ŝ (no images yet)
        drafts.append(draft)
    
    # Phase 4: Image grounding + audit
    final_skills = []
    for draft in drafts:
        grounded = ground_keyframes(draft, trajectory_pool, meta_skill)
        audited = audit_skill(grounded, meta_skill)
        final_skills.append(audited)
    
    return final_skills
```

---

## 关键超参数与配置

| 配置项 | 说明 |
|---|---|
| OSWorld 交互上限 | 20 步/任务 |
| 技能来源 | OpenCUA 公开非测试轨迹（与评测任务分离） |
| 评估模型 | Gemini 3.1 Pro, Gemini 3 Flash, Qwen3-VL-235B, GLM-5V, Kimi-K2.6, Qwen3-VL-8B |
| 关键帧视图类型 | full_frame, focus_crop, before, after |
| Branch Stage 1 | 选择性加载视觉参考（防止上下文超载） |
| Branch Stage 2 | 返回结构化决策 tuple（不直接返回 action） |

---

*Summary generated from arxiv 2605.13527 LaTeX source using read-arxiv-paper skill.*
