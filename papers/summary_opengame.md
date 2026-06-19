# OpenGame: Open Agentic Coding for Games

**Paper:** arxiv 2604.18394
**URL:** https://arxiv.org/abs/2604.18394
**Authors:** Yilei Jiang, Jinyuan Hu, Qianyin Xiao, Yaozhi Zheng, Ruize Ma, Kaituo Feng, Jiaming Han, Tianshuo Peng, Kaixuan Fan, Manyuan Zhang, Xiangyu Yue | CUHK MMLab

---

## 论文目标

游戏开发需要游戏引擎、实时循环、跨文件状态管理的协同。LLMs 在端到端创建可玩完整游戏时经常失败，表现为：
1. **逻辑不一致**：丢失全局状态追踪，产生冻结、不终止或无法实现关键机制的项目
2. **引擎知识缺口**：错误使用或绕过框架原生 API，绕过 Phaser 3 的物理/场景/事件系统
3. **跨文件不一致**：资产键不匹配、场景接线错误、配置字段缺失或初始化顺序错误

核心问题：**如何让 code agent 从自然语言设计规格生成完整可玩的游戏？**

---

## 核心思想

**OpenGame**：首个开源 agentic 框架，专为端到端网页游戏创建设计。

**Game Skill = 可复用能力**，由两个组件构成：

```
Game Skill = Template Skill + Debug Skill

Template Skill：
  - 初始：单一游戏无关元模板 M₀
  - 进化：从经验中增长项目骨架库 L
  - 结果：形成 5 个专业模板家族（重力侧视图、俯视连续运动、离散网格逻辑、路径波动态、UI驱动）

Debug Skill：
  - 维护活态调试协议 P
  - 记录：(error_signature, root_cause, verified_fix)
  - 泛化：高频错误模式 → 可复用规则
  - 预执行验证 + 执行后修复
```

---

## 三大挑战与解法

| 挑战 | 解法 |
|------|------|
| **端到端游戏生成** | 六阶段结构化工作流（分类→脚手架→设计→资产→实现→验证） |
| **多文件一致性** | Template Skill（项目骨架）+ Debug Skill（累积调试协议） |
| **交互式软件评估** | OpenGame-Bench：动态可玩性评估（BH/VU/IA 三维度） |

---

## 方法一：GameCoder-27B 训练管道

```
基础模型：Qwen3.5-27B

Stage 1: Continual Pre-Training (CPT)
  - 数据：GitHub Phaser/JavaScript/TypeScript 游戏仓库
  - 官方文档 + 社区教程
  - 目标：建立游戏循环、物理系统、资产使用、状态管理先验

Stage 2: Supervised Fine-Tuning (SFT)
  - GPT-5.1 生成复杂多步游戏设计提示
  - MiniMax-2.5 生成高质量目标代码
  - 目标：将抽象创意意图转换为具体代码结构

Stage 3: Reinforcement Learning (RL)
  - 执行反馈驱动的单元测试
  - 单文件游戏逻辑和目标功能模块
  - 目标：在进入完整多文件项目前强化确定性可执行逻辑
```

---

## 方法二：六阶段 Agent 工作流

```
1. Initialization & Classification
   - Physics-First Classification：根据物理约束和空间机制分类（而非模糊的 genre 标签）
   - classify-game-type 工具返回 {archetype}

2. Scaffolding & Design Generation
   - run_shell_command 复制对应 archetype 模板到工作区
   - generate-gdd 生成技术 Game Design Document (GDD)
   - todo_write 工具分解为文件级操作

3. Multimodal Asset Synthesis
   - generate-game-assets：AI 多模态生成背景、角色动画、静态物品、音频
   - generate-tilemap：ASCII 布局 → 结构化 JSON tilemap
   - 记录精确的 texture 和 asset keys

4. Context-Aware Code Implementation
   - Three-Layer Reading Strategy：
     Layer 1: API summary（模板系统）
     Layer 2: targeted source file（待修改文件）
     Layer 3: implementation guide（最后加载，最大即时显著性）
   - Template Method Pattern：复制模板文件 + override designated hook methods

5. Verification & Self-Correction
   - debug_protocol.md 静态自检
   - npm run build + npm run test（headless browser 执行）
   - 迭代修复直到获得可玩游戏
```

---

## 方法三：Agent Evolution with Game Skills

### Template Skill

```
M₀（初始元模板）：
  - 单一游戏无关项目骨架
  - 定义：项目布局、初始化、资产加载、场景循环、配置接口
  - 不假设任何 genre、物理机制或游戏机制

L（进化模板库）：
  - 任务完成后识别可重用代码片段：
    (i) 跨游戏稳定
    (ii) 广泛有用
    (iii) 安全可复用
  - 抽象为可重用模板单元 + 约束，合并到 L
  - 随时间演化为 5 个专业模板家族
```

### Debug Skill

```
P（活态调试协议）：
  - 每次失败记录结构化条目：(error_signature, root_cause, verified_fix)
  - 高频不一致类别的预执行验证：
    * 资产键不匹配
    * 缺失配置字段
    * 无效场景转换

协议更新规则：
  - 失败模式重现 → 泛化为可复用规则
  - 新失败出现 → 协议扩展新条目
  - 调试知识累积 + 持久化，不增加 prompt 复杂度
```

### Game Skill 执行算法

```
输入: x (用户规格), M₀ (元模板), L (模板库), P (调试协议)

1. 选择合适的模板家族 T ∈ L（初始化为 M₀）
2. 实例化 T 脚手架项目骨架 y
3. 在 y 的扩展点内生成游戏特定内容（条件于 x）

repeat 直到收敛:
   4. 运行验证和执行（build, test, run），由 P 引导
   5. if 失败:
      - 用 P 诊断失败并修复 y
      - 如果模式新颖，追加验证过的 (signature, cause, fix) 条目到 P

optional: 从 y 提取可重用片段并合并到 L

return y (可运行游戏项目)
```

---

## OpenGame-Bench 评估

### 评估维度

| 维度 | 指标 | 说明 |
|------|------|------|
| **Build Health (BH)** | [0,100] | 编译和运行时稳定性；捕获损坏依赖、JS 运行时异常、静默网络失败 |
| **Visual Usability (VU)** | [0,100] | 像素级启发式（帧熵 + 运动检测）+ VLM 评判；奖励连贯、动画化、可见可交互内容 |
| **Intent Alignment (IA)** | [0,100] | VLM 评判器对照结构化需求规格的加权通过率 |

### Benchmark

- **150 个任务**：5 种游戏类型（platformers, top-down shooters, puzzle games, arcade classics, strategy）
- **评测约束**：明确要求使用 Phaser 3 框架
- **评估方式**：headless browser 执行 + VLM 评判
- **随机性**：每个任务 3 次不同随机种子

---

## 实验结果

### 主结果

| Category | System / Model | BH | VU | IA |
|----------|---------------|-----|-----|-----|
| Direct LLMs (Open) | Qwen-3.5-Max | 51.8 | 35.5 | 38.9 |
| | MiniMax m2.5 | 39.7 | 39.3 | 31.8 |
| | DeepSeek V3.2 | 57.0 | 38.9 | 33.5 |
| Direct LLMs (Closed) | Claude Sonnet 4.6 | 58.5 | 50.8 | 50.3 |
| | GPT-5.1 | 57.4 | 52.9 | 49.4 |
| | Gemini 3.1 Pro | 53.6 | 60.2 | 42.1 |
| Agentic Frameworks | qwen-code (w/ Claude) | 63.2 | 54.3 | 57.8 |
| | Cursor (w/ Claude) | 66.8 | 61.4 | 58.9 |
| **Ours** | **OpenGame (w/ GameCoder-27B)** | **63.9** | **57.0** | **54.1** |
| **Ours** | **OpenGame (w/ Claude Sonnet 4.6)** | **72.4** | **67.2** | **65.1** |

**关键洞察**：
- OpenGame (w/ Claude) 超越最强基线 Cursor (w/ Claude)：BH +5.6, VU +5.8, IA +6.2
- Intent Alignment 增益最大 (+6.2)：结构化规划 + 模板脚手架 + 迭代验证更好保留用户指定机制
- GameCoder-27B 超越所有直接开源 + 闭源 LLM 基线

### Ablation I：训练管道

| Model Stage | BH | VU | IA |
|-------------|-----|-----|-----|
| Base Model (Qwen-3.5-27B) | 62.8 | 53.8 | 49.8 |
| + CPT | 63.2 | 54.7 | 50.6 |
| + CPT + SFT | 63.5 | 55.7 | 52.5 |
| + CPT + SFT + RL (Full) | 63.9 | 57.0 | 54.1 |

### Ablation II：Agent 工作流机制

| Agent Configuration | BH | VU | IA |
|--------------------|-----|-----|-----|
| OpenGame (Full Workflow) | **72.4** | **67.2** | **65.1** |
| w/o Hook-Driven Implementation | 62.3 | 57.6 | 53.5 |
| w/o Three-Layer Reading | 67.8 | 61.9 | 56.5 |
| w/o Physics-First Classification | 70.2 | 64.6 | 61.6 |

**关键洞察**：Template Method Pattern（Hook-Driven）影响最大，禁用后 BH -10.1, IA -11.6

### Ablation III：Agent Evolution

| Template Architecture | Debugging Strategy | BH | VU | IA |
|----------------------|-------------------|-----|-----|-----|
| Static Skeleton (M₀) | Static Rule Checklist | 60.5 | 54.8 | 51.2 |
| Static Skeleton (M₀) | Full Living Protocol (P) | 65.4 | 59.2 | 56.3 |
| Partial (2 Families) | Static Rule Checklist | 63.1 | 57.3 | 53.8 |
| Full (5 Families) | Static Rule Checklist | 66.3 | 60.7 | 57.9 |
| Full (5 Families) | Post-Execution Fixes Only | 69.5 | 63.8 | 61.4 |
| Full (5 Families) | Full Living Protocol (P) | **72.4** | **67.2** | **65.1** |

### 迭代调试效果

- T=0（零样本）：BH = 58.4
- T=1-3：陡峭增益
- T=5：性能趋于平稳
- **关键发现**：有界迭代修复是可靠长时域游戏生成的关键

### 风格分析

| Genre | OpenGame IA | Cursor IA |
|-------|-------------|-----------|
| Platformers | **76.8** | 68.2 |
| Top-Down Shooters | **71.4** | 62.1 |
| Arcade Classics | 66.5 | 61.8 |
| Strategy | 58.2 | 52.4 |
| Puzzle/UI | 52.6 | 48.3 |

**洞察**：OpenGame 在物理中心和空间接地环境中最强；策略/Puzzle 逻辑状态管理更抽象，silent failures 更难检测。

---

## 与 GEPA、MLEvolve、SkillOpt、MMSkills 的比较

| 维度 | GEPA | MLEvolve | SkillOpt | MMSkills | **OpenGame** |
|------|------|----------|----------|----------|--------------|
| 优化对象 | 模块 prompt | 搜索策略 + 记忆 | Skill document | 多模态 Skill Package | **游戏生成框架** |
| 核心创新 | Pareto selection | Graph MCGS + memory | Validation gate | Branch Loading + State Cards | **Template Skill + Debug Skill** |
| 技能表示 | 文本 | 记忆 + 图 | 文本 + edits | 文本 + 状态卡 + 关键帧 | **模板骨架 + 调试协议** |
| 利用方式 | 直接插入 | 图搜索 | 验证后接受 | Branch Loading | **结构化工作流 + 迭代验证** |
| 反馈来源 | trace + 评估 | 执行 + memory | Rollout + gate | 公开轨迹 + 视觉 grounding | **Build + Test + Runtime 反馈** |
| 生成方式 | 人类编写/一次生成 | 演化搜索 | 优化训练 | 五阶段 Generator | **经验累积 + 协议更新** |
| 基础模型 | 通用 LLM | 通用 LLM | 通用 LLM + 验证 | 通用 VL LLM | **专用 GameCoder-27B** |

---

## 与 scenegen_agent 项目的联系

OpenGame 的 **Template Skill + Debug Skill** 机制与 Branch Loading 的设计思路有相通之处：

| OpenGame | Branch Loading |
|----------|----------------|
| Template Skill 选择合适的模板家族 | Stage 1 选择合适的 state cards |
| Debug Skill 积累错误模式 | Stage 2 对齐参考证据与当前状态 |
| 活态协议 P 随经验进化 | Skill 中的 K（关键帧）随录制累积 |
| 六阶段结构化工作流 | 主 Agent 收到结构化决策 tuple |
| 迭代验证直到可玩 | 迭代修复直到完成验证 |

### 迁移到 scenegen 的可能方向

1. **Template Skill → Planning Skill Library**：
   - 初始元模板 M₀ = 通用驾驶规划流程
   - 进化模板库 = collision-prone 场景专用规划策略
   - 对应 scenegen 从 collision logs 自动生成 planning rules

2. **Debug Skill → Verification Protocol**：
   - 活态调试协议 P = collision 修复知识库
   - 记录：(collision_signature, root_cause, verified_fix)
   - 预执行验证：检查边界框、对齐状态、速度阈值

3. **Hook-Driven Implementation → Scenario Hooks**：
   - Template Method Pattern → scenario-specific override hooks
   - 主规划流程固定，场景特定逻辑通过 hooks 注入

4. **Three-Layer Reading → Progressive Evidence Loading**：
   - 渐进式加载证据（API summary → source → guide）
   - 对应 Branch Loading 的两阶段选择性加载

5. **迭代验证 → 验证驱动的规划调整**：
   - Build/Test/Run 循环 → planning/verification/rollback 循环
   - 每次规划后验证预期状态，不满足则回滚

---

## 关键公式

- **Game Skill**: $M = (T, P)$
  - $T$: Template Skill (骨架库)
  - $P$: Debug Skill (调试协议)
- **Template Evolution**: $L \leftarrow L \cup \text{extract\_reusable}(y)$
- **Debug Update**: $P \leftarrow P \cup \{(sig, cause, fix)\}$
- **OpenGame Execution**: $y = \text{GameSkill}(x, M_0, L, P)$

---

## 详细算法流程

### Game Skill 执行

```python
def game_skill_execution(x, M0, L, P):
    # Select template family
    T = select_template(L, x)  # initialized as M0
    
    # Scaffold project
    y = instantiate(T)
    
    # Generate game-specific content
    y = generate_content(y, x)
    
    # Iterative verification and repair
    for iteration in range(max_iterations):
        # Run verification (build, test, run)
        if verify(y, P):
            return y
        
        # Diagnose failure
        failure = diagnose(y, P)
        
        # Repair
        y = repair(y, failure)
        
        # Update protocol if new pattern
        if is_new_pattern(failure):
            P.add(failure)
    
    # Optional: extract reusable fragments
    L = extract_and_merge(y, L)
    
    return y
```

### 六阶段工作流

```python
def six_phase_workflow(user_spec):
    # Phase 1: Classification
    archetype = classify_game_type(user_spec)  # Physics-First
    
    # Phase 2: Scaffolding
    scaffold_project(archetype)  # Copy template family
    gdd = generate_gdd(user_spec, archetype)  # Technical design doc
    todos = decompose_todos(gdd)
    
    # Phase 3: Asset Synthesis
    assets = generate_game_assets(gdd.asset_registry)
    tilemaps = generate_tilemap(gdd.ascii_maps)
    
    # Phase 4: Config & Implementation
    merge_config(gdd.params)
    three_layer_reading()  # API summary → source → guide
    implement_with_hooks(todos)  # Template Method Pattern
    
    # Phase 5: Verification
    static_check(debug_protocol)
    run_build_and_test()
    iterate_until_playable()
    
    return final_project
```

---

## 关键超参数与配置

| 配置项 | 说明 |
|--------|------|
| Benchmark | 150 个任务，5 种游戏类型 |
| 框架约束 | 明确要求 Phaser 3 |
| 评估维度 | BH, VU, IA 各 [0,100] |
| 随机性 | 每任务 3 次随机种子 |
| 模板家族数 | 5（进化得出，非预设） |
| 调试迭代 | T ∈ {0, 1, 2, 3, 4, 5} |
| 基础模型 | Qwen3.5-27B + 领域持续预训练 + SFT + RL |

---

## 核心贡献总结

1. **OpenGame**：首个开源 agentic 框架，专为端到端网页游戏创建
2. **Game Skill**：Template Skill（稳定脚手架）+ Debug Skill（累积错误修复）
3. **GameCoder-27B**：三阶段训练管道（持续预训练 + SFT + 执行 RL）
4. **OpenGame-Bench**：超越静态单元测试的动态可玩性评估

---

*Summary generated from arxiv 2604.18394 LaTeX source.*
