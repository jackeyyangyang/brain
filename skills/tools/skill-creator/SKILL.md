---
name: skill-creator
description: 创建新 skill、修改改进现有 skill、并测量 skill 性能。当用户想要从零创建 skill、编辑或优化现有 skill、运行评测来测试 skill、进行带方差分析的 skill 性能基准测试、或优化 skill 描述以提高触发准确率时，启用此 skill。
---

# Skill Creator

用于创建新 skill 并持续迭代改进的 skill。

从高层看，创建 skill 的流程如下：

- 确定 skill 要做什么、大概怎么做
- 写出 skill 初稿
- 设计几个测试 prompt，在 claude-with-access-to-the-skill 上跑一遍
- 帮助用户从定性和定量两个角度评估结果
  - 在后台运行期间，如果没有现成的评测，可以起草一些定量评测（如果有现成的，可以直接用或按需修改）。然后向用户解释这些评测（或已有评测的解释）
  - 用 `eval-viewer/generate_review.py` 脚本向用户展示结果，也让他们看定量指标
- 根据用户对结果的反馈重写 skill（以及定量基准测试中发现的明显缺陷时）
- 重复直到满意
- 扩大测试集，在更大规模上再跑

你在使用这个 skill 时的工作，是弄清楚用户处于这个流程的哪个阶段，然后帮他们推进。所以比如用户说"我想做一个 X 的 skill"，你可以帮他：缩小范围、写初稿、写测试用例、确定评测方式、跑所有 prompt、然后迭代。

另一方面，也许用户已经有一个初稿了。这种情况下可以直接进入评测/迭代环节。

当然，要灵活——如果用户说"不需要跑一堆评测，一起聊聊就行"，那就直接聊。

然后在 skill 完成之后（顺序也是灵活的），还可以运行 skill description improver，这是个独立的脚本，用来优化 skill 的触发准确率。

## 与用户沟通

Skill Creator 的用户可能对编程术语的熟悉程度差异很大。现在有一种趋势：Claude 的能力激发了管道工人、家长和祖父母开始打开终端、搜索"怎么装 npm"。另一方面，大多数用户可能相当熟悉电脑。

所以请注意上下文线索，理解怎么措辞！默认情况下：
- "evaluation"和"benchmark"是边界词，但可以用
- 对于"JSON"和"assertion"，除非看到用户有认真的线索表明他们知道这些词，否则要解释一下

有疑问时可以简要解释术语，如果你不确定用户是否理解，放心去澄清。

## 创建 Skill

### 捕获意图

从理解用户的意图开始。当前对话可能已经包含了用户想捕获的工作流（比如用户说"把这个转成 skill"）。如果是，先从对话历史中提取答案——工具是什么、步骤顺序、用户的修正、观察到的输入/输出格式。用户可能需要补充一些缺口，应该在继续下一步前确认。

1. 这个 skill 应该让 Claude 做什么？
2. 应该在什么时候触发？（用户的哪些表述/上下文）
3. 预期的输出格式是什么？
4. 要不要设置测试用例来验证 skill 可用？具有客观可验证输出的 skill（文件转换、数据提取、代码生成、固定工作流步骤）受益于测试用例。具有主观输出的 skill（写作风格、艺术）通常不需要。

### 访谈与调研

主动询问边界情况、输入/输出格式、示例文件、成功标准和依赖项。等这些都理清了再写测试 prompt。

检查可用的 MCP——如果对调研有用（搜索文档、找类似 skill、查阅最佳实践），如果有子代理就并行研究，否则就内联进行。带好上下文去见用户，减少他们的负担。

### 写 SKILL.md

根据用户访谈，填充这些组件：

- **name**：Skill 标识符
- **description**：何时触发、做什么。这是主要的触发机制——要同时包含 skill 做什么和具体的触发上下文。所有"何时使用"信息都写在这里，不写在正文。**注意**：目前 Claude 有"触发不足"的倾向——在有用的情况下不会使用 skill。为了解决这个问题，请把 skill 描述写得有"推动感"一些。比如不要说"如何构建一个简单的快速仪表盘来显示内部 Anthropic 数据"，而是说"如何构建一个简单的快速仪表盘来显示内部 Anthropic 数据。确保在用户提到仪表盘、数据可视化、内部指标、或想要展示任何类型公司数据时使用此 skill，即使他们没有明确要求'dashboard'。"
- **compatibility**：必需的工具、依赖项（可选，很少需要）
- **剩下的就是 skill 本身 :)**

### Skill 写作指南

#### Skill 的组成结构

```
skill-name/
├── SKILL.md（必需）
│   ├── YAML frontmatter（name、description 必需）
│   └── Markdown 指令
└── 打包资源（可选）
    ├── scripts/    - 用于确定性/重复性任务的可执行代码
    ├── references/  - 按需加载到上下文的文档
    └── assets/     - 输出中使用的文件（模板、图标、字体等）
```

#### 渐进式披露

Skill 使用三层加载系统：
1. **元数据**（name + description）——始终在上下文中（~100 词）
2. **SKILL.md 正文**——skill 触发时在上下文中（理想 < 500 行）
3. **打包资源**——按需（无限，脚本可以执行而不加载）

这些字数是近似值，感觉需要可以突破。
**关键模式：**
- SKILL.md 保持在 500 行以内；如果快到上限了，加一层分级结构并给出清晰的指针，告诉使用该 skill 的模型接下来去哪继续
- 从 SKILL.md 引用文件时，给出何时读取的指导
- 对于 >300 行的大参考文件，包含目录

**按领域组织**：当 skill 支持多个领域/框架时，按变体组织：
```
cloud-deploy/
├── SKILL.md（工作流 + 选择）
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```
Claude 只读相关的参考文件。

#### 勿扰原则

不用说，skill 不能包含恶意软件、漏洞利用代码或任何可能危及系统安全的内容。Skill 的内容如果被描述出来，不应该在意图上让用户惊讶。不要配合创建误导性的 skill 或旨在促进未授权访问、数据外泄或其他恶意活动的 skill。像"扮演 XYZ 角色"这种是可以的。

#### 写作模式

指令优先使用祈使语气。

**定义输出格式**——可以这样写：
```markdown
## 报告结构
始终使用这个精确模板：
# [标题]
## 执行摘要
## 关键发现
## 建议
```

**示例模式**——包含示例很有用。可以这样格式化（但如果"输入"和"输出"在示例中，可以稍微变通）：
```markdown
## Commit message 格式
**示例 1：**
输入：添加了带 JWT token 的用户认证
输出：feat(auth): implement JWT-based authentication
```

### 写作风格

尽量解释模型为什么事情重要，而不是用老派的 MUST 大写来压人。用心理理论，尽量让 skill 通用而不是非常窄地针对特定例子。先写初稿，然后用新眼光看一遍再改进。

### 测试用例

写完 skill 初稿后，想出 2-3 个真实的测试 prompt——真实用户实际会说的那种。分享给用户：[不一定用这个确切的语言]"这里有几个我想跑的测试用例。看起来对吗，想加更多吗？"然后跑它们。

把测试用例保存到 `evals/evals.json`。先不写断言——只有 prompt。等下一步再写断言，跑的时候在后台进行。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "用户的任务 prompt",
      "expected_output": "预期结果描述",
      "files": []
    }
  ]
}
```

参见 `references/schemas.md` 获取完整 schema（包括后面要加的 `assertions` 字段）。

## 运行与评测测试用例

本节是一个连续的过程——不要半途停下。不要用 `/skill-test` 或任何其他测试 skill。

把结果放到 ` -workspace/` 里作为 skill 目录的兄弟。在那里，按迭代组织（`iteration-1/`、`iteration-2/` 等），在每个迭代下，每个测试用例一个目录（`eval-0/`、`eval-1/` 等）。不要一开始就全部创建——边走边建。

### 步骤 1：同时启动所有运行（with-skill 和 baseline）

对于每个测试用例，在同一轮同时启动两个子代理——一个有 skill，一个没有。这很重要：不要先跑 with-skill 再回头跑 baseline。一次全部启动，这样都差不多同时结束。

**With-skill 运行：**

```
执行这个任务：
- Skill 路径：<path-to-skill>
- 任务：<eval prompt>
- 输入文件：<eval 文件，如果有的话，或"无">
- 保存输出到：<workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- 要保存的输出：<用户关心的内容——比如".docx 文件"、"最终的 CSV">
```

**Baseline 运行**（同样的 prompt，但 baseline 取决于上下文）：
- **创建新 skill**：完全没有 skill。同样的 prompt，没有 skill 路径，保存到 `without_skill/outputs/`。
- **改进现有 skill**：旧版本。在编辑前，先快照 skill（`cp -r /skill-snapshot/`），然后让 baseline 子代理指向快照。保存到 `old_skill/outputs/`。

给每个测试用例写一个 `eval_metadata.json`（断言可以暂时为空）。给每个 eval 一个描述性名称，基于它在测什么——不要只是"eval-0"。目录也用这个名字。如果这个迭代用了新的或修改过的 eval prompt，给每个新 eval 目录都创建这些文件——不要假设它们会从上一次迭代继承。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "用户的任务 prompt",
  "assertions": []
}
```

### 步骤 2：在运行进行中起草断言

不要只是等运行结束——可以高效利用这段时间。为每个测试用例起草定量断言，并向用户解释。如果 `evals/evals.json` 中已有断言，审查并解释它们在检查什么。

好的断言是客观可验证的，并且有描述性名称——它们在基准测试查看器中应该能让人一眼就看出每个在检查什么。具有主观输出的 skill（写作风格、设计质量）更适合定性评估——不要把断言强加给需要人类判断的事情。

在准备好后将 `eval_metadata.json` 文件和 `evals/evals.json` 更新为断言。同时向用户解释他们在查看器里会看到什么——既有定性输出，也有定量基准。

### 步骤 3：运行时捕获计时数据

每个子代理任务完成时，你会收到包含 `total_tokens` 和 `duration_ms` 的通知。立即把这个数据保存到运行目录的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是捕获这个数据的唯一机会——它通过任务通知过来，不会持久化到别处。处理每个通知要趁热，不要想着批量来。

### 步骤 4：评分、聚合、启动查看器

所有运行结束后：

1. **给每个运行评分**——启动一个评分子代理（或内联评分），读取 `agents/grader.md` 并对每个断言评估输出。将结果保存到每个运行目录的 `grading.json`。grading.json 的 expectations 数组必须使用字段 `text`、`passed` 和 `evidence`（不是 `name`/`met`/`details` 或其他变体）——查看器依赖这些确切的字段名。对于可以编程检查的断言，写个脚本跑而不是用眼睛看——脚本更快、更可靠、可以在迭代间复用。

2. **聚合为基准**——从 skill-creator 目录运行聚合脚本：
   ```bash
   python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
   ```
   这会生成 `benchmark.json` 和 `benchmark.md`，包含每个配置 pass_rate、时间和 tokens，带 mean ± stddev 和 delta。如果手动生成 benchmark.json，参见 `references/schemas.md` 获取查看器期望的确切 schema。
   把每个 with_skill 版本放在其 baseline 对应版本前面。

3. **做一轮分析师**——读取基准数据，提炼出聚合统计可能隐藏的模式。参见 `agents/analyzer.md` 的"分析基准测试结果"部分——要看的东西比如：总是通过的断言，不管有没有 skill（无区分力）、高方差 eval（可能不稳定）和时间/token 权衡。

4. **启动查看器**，同时展示定性输出和定量数据：
   ```bash
   nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
   ```
   对于第 2 轮及以后，还要传 `--previous-workspace /iteration- `。

   **Cowork / 无头环境**：如果 `webbrowser.open()` 不可用或环境没有显示器，用 `--static ` 写一个独立 HTML 文件代替启动服务器。用户点击"提交所有反馈"后，反馈会下载为 `feedback.json` 文件。下载后，把 `feedback.json` 复制到 workspace 目录，供下一轮迭代拾取。

   注意：请用 generate_review.py 创建查看器；不需要写自定义 HTML。

5. **告诉用户**类似："我在浏览器里打开了结果。有两个标签页——'Outputs' 让你逐个点击每个测试用例并留下反馈，'Benchmark' 显示定量比较。完成后回到这里告诉我。"

### 查看器里用户看到什么

"Outputs"标签页一次显示一个测试用例：
- **Prompt**：给出的任务
- **Output**：skill 生成的文件，尽可能内联渲染
- **Previous Output**（第 2 轮及以后）：折叠区域，显示上一轮迭代的输出
- **Formal Grades**（如果运行了评分）：折叠区域，显示断言 pass/fail
- **Feedback**：一个输入框，输入时自动保存
- **Previous Feedback**（第 2 轮及以后）：他们上次的评论，显示在输入框下方

"Benchmark"标签页显示统计摘要：每个配置的 pass rates、时间、token 使用，带 per-eval 细分和分析师观察。

导航通过 prev/next 按钮或方向键。完成后，点击"提交所有反馈"，将所有反馈保存到 `feedback.json`。

### 步骤 5：读取反馈

用户告诉你完成后，读取 `feedback.json`：

```json
{
  "reviews": [
    {"run_id": "eval-0-with_skill", "feedback": "图表缺少坐标轴标签", "timestamp": "..."},
    {"run_id": "eval-1-with_skill", "feedback": "", "timestamp": "..."},
    {"run_id": "eval-2-with_skill", "feedback": "完美，喜欢这个", "timestamp": "..."}
  ],
  "status": "complete"
}
```

空反馈意味着用户觉得没问题。把改进重点放在用户有具体抱怨的测试用例上。

完成后，关闭查看器服务器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改进 Skill

这是循环的核心。你已经跑了测试用例，用户已经审阅了结果，现在需要根据反馈让 skill 变得更好。

### 如何思考改进

1. **从反馈中泛化。** 大局是：我们正在尝试创建可以使用无数次（也许 literally，可能更多）跨许多不同 prompt 的 skill。你和用户在这里通过几个例子反复迭代是因为这样更快。用户对这些例子了如指掌，评估新输出也很快。但如果你们共同开发的 skill 只适用于那些例子，它就没用了。与其做些花哨的、过拟合的修改，或者压抑性地限制性 MUSTS，如果有什么固执的问题，可以尝试分支出去，用不同的比喻，或者推荐不同的工作模式。试试相对便宜，也许你会发现很棒的东西。

2. **保持 prompt 精简。** 删除没有发挥作用的东
   西。确保读一下 transcripts，不只是最终输出——如果看起来 skill 让模型浪费了大量时间做无产出的事情，可以尝试删掉让模型那样做的 skill 部分，看看会发生什么。

3. **解释为什么。** 尽力解释你要求做的每件事的**为什么**。当今的 LLM 很聪明。它们有很好的心理理论，当给了一个好的 harness，可以超越死板的指令真正发挥作用。即使用户的反馈简洁或沮丧，也要试着真正理解任务、理解用户为什么写他们写的内容、理解他们实际写了什么，然后把这种理解传递到指令中。如果你发现自己写了很多大写的 ALWAYS 或 NEVER，或者使用超级死板的结构，这是个黄旗——如果可能的话，重新框架并解释推理，让模型理解你要求做的事情为什么重要。这是更人性化、更有力、更有效的方法。

4. **寻找测试用例间的重复工作。** 读一下测试运行的 transcripts，注意是否所有子代理都独立写了类似的辅助脚本或对某事采取了相同的多步方法。如果所有 3 个测试用例都导致子代理写了 `create_docx.py` 或 `build_chart.py`，这是强烈信号，表明 skill 应该打包那个脚本。写一次，放到 `scripts/`，告诉 skill 用它。这为每次未来调用节省了重新发明轮子。

这个任务相当重要（我们正在这里尝试创造每年数十亿美元的经济价值！），你的思考时间不是瓶颈；好好琢磨一下。我建议写个修订草案，然后用新的眼光看一遍再改进。真的尽力进入用户的头脑，理解他们想要什么、需要什么。

### 迭代循环

改进 skill 后：

1. 把所有测试用例重新跑一轮，进入新的 `iteration- /` 目录，包括 baseline 运行。如果在创建新 skill，baseline 始终是 `without_skill`（无 skill）——跨迭代保持不变。如果在改进现有 skill，用你的判断决定 baseline 什么有意义：用户带来的原始版本，还是上一轮迭代。
2. 用 `--previous-workspace` 指向上一轮迭代，启动审阅者
3. 等用户审阅完告诉你
4. 读新反馈，再改进，再重复

一直继续直到：
- 用户说满意了
- 反馈都是空的（一切看起来都好）
- 没有取得有意义的进展

---

## 高级：盲对比

对于想要更严格比较两个 skill 版本的情况（比如用户问"新版本真的更好吗？"），有盲对比系统。读 `agents/comparator.md` 和 `agents/analyzer.md` 获取详情。基本思路是：把两个输出交给一个独立 agent，不告诉它哪个是哪个，让它评判质量。然后分析为什么赢家赢了。

这是可选的，需要子代理，大多数用户不需要。人类审阅循环通常就足够了。

---

## 描述优化

SKILL.md frontmatter 的 description 字段是决定 Claude 是否调用 skill 的主要机制。创建或改进 skill 后，提供优化 description 以提高触发准确率。

### 步骤 1：生成触发评测查询

创建 20 个评测查询——混合 should-trigger 和 should-not-trigger。保存为 JSON：

```json
[
  {"query": "用户的 prompt", "should_trigger": true},
  {"query": "另一个 prompt", "should_trigger": false}
]
```

查询必须是真实的，是 Claude Code 或 Claude.ai 用户实际会输入的。不是抽象请求，而是具体、详细、真实的请求。比如包含文件路径、关于用户工作或个人情况的背景信息、列名和值、公司名、URL。一点点背景故事。用不同长度混合，关注边界情况而不是让它们明显（用户会有机会签字确认）。

不好：`"格式化这些数据"`、`"从 PDF 提取文本"`、`"创建图表"`

好：`"我老板刚给我发了一个 xlsx 文件（在我下载里，叫类似'Q4 销售 final FINAL v2.xlsx'），她让我加一列显示利润率百分比。收入在 C 列，成本在 D 列我记得"`

对于 **should-trigger** 查询（8-10 个），考虑覆盖率。想要同一意图的不同措辞——有些正式，有些随意。包括用户没有明确命名 skill 或文件类型但明显需要它的情况。加入一些不常见的用例，以及 skill 与另一个竞争但应该胜出的情况。

对于 **should-not-trigger** 查询（8-10 个），最有价值的是 near-miss——与 skill 共享关键词或概念但实际需要不同东西的查询。考虑相邻领域、歧义措辞（简单的关键词匹配会触发但不应该），以及查询涉及 skill 做的事但在另一个工具更合适的上下文中的情况。

关键要避免的：不要让 should-not-trigger 查询明显无关。"把 fibonacci 函数"作为 PDF skill 的负面测试太容易了——它什么也没测。负面用例应该真正棘手。

### 步骤 2：与用户一起审阅

用 HTML 模板向用户展示评测集：

1. 从 `assets/eval_review.html` 读取模板
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` → JSON 数组（不加分号——是 JS 变量赋值）
   - `__SKILL_NAME_PLACEHOLDER__` → skill 名称
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → skill 当前 description
3. 写到一个临时文件（比如 `/tmp/eval_review_.html`），用 `open /tmp/eval_review_.html` 打开
4. 用户可以编辑查询、切换 should-trigger、添加/删除条目，然后点击"导出评测集"
5. 文件下载到 `~/Downloads/eval_set.json`——检查下载文件夹里最新版本（可能有多个，比如 `eval_set (1).json`）

这一步很重要——糟糕的评测查询导致糟糕的 description。

### 步骤 3：运行优化循环

告诉用户："这需要一些时间——我会在后台运行优化循环，定期检查进度。"

把评测集保存到 workspace，然后在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

用当前会话驱动的 model ID，这样触发测试与用户实际体验匹配。

运行时，定期 tail 输出，给用户更新它在第几轮迭代、分数量级看起来怎么样。

这自动处理完整的优化循环。它把评测集分成 60% 训练和 40% 保留测试，评估当前 description（在每个查询上跑 3 次得到可靠的触发率），然后调用 Claude 根据失败情况提出改进。它在训练和测试上重新评估每个新 description，迭代最多 5 次。完成后，在浏览器里打开 HTML 报告，显示每轮迭代的结果，并返回带 `best_description` 的 JSON——按测试分数（不是训练分数）选择以避免过拟合。

### Skill 触发如何工作

理解触发机制有助于设计更好的评测查询。Skill 以 name + description 出现在 Claude 的 `available_skills` 列表中，Claude 根据 description 决定是否查阅 skill。需要知道的重要一点：Claude 只在它不能轻易处理的任务上查阅 skill——简单、一步到位的查询比如"读这个 PDF"可能不会触发 skill，即使 description 完全匹配，因为 Claude 可以直接用基本工具处理。复杂、多步或专业化的查询，当 description 匹配时，会可靠地触发 skill。

这意味着你的评测查询应该足够实质，让 Claude 真正受益于查阅 skill。"读文件 X"这种简单查询是糟糕的测试用例——不管 description 质量如何都不会触发 skill。

### 步骤 4：应用结果

从 JSON 输出中取 `best_description`，更新 skill 的 SKILL.md frontmatter。先给用户展示前/后，并报告分数量级。

---

## Claude.ai 特定说明

在 Claude.ai 中，核心工作流相同（初稿 → 测试 → 审阅 → 改进 → 重复），但因为 Claude.ai 没有子代理，一些机制会变。以下是需要适应的部分：

**跑测试用例**：没有子代理意味着不能并行执行。对于每个测试用例，读 skill 的 SKILL.md，然后按其指令完成任务 prompt 自己跑。一次一个。这不如独立子代理严格（你写了 skill 也在跑它，你有完整上下文），但人类审阅步骤补偿了。跳过 baseline 运行——只用 skill 完成请求的任务。

**审阅结果**：如果不能打开浏览器（比如 Claude.ai 的 VM 没有显示器，或者在远程服务器上），完全跳过浏览器审阅者。相反，在对话中直接呈现结果。对于每个测试用例，显示 prompt 和输出。如果输出是一个用户需要看的文件（比如 .docx 或 .xlsx），保存到文件系统，告诉他们放在哪以便下载检查。Inline 询问反馈："这看起来怎么样？有什么想改的吗？"

**基准测试**：跳过定量基准测试——它依赖没有子代理就没有意义的 baseline 比较。专注于用户的定性反馈。

**迭代循环**：相同——改进 skill，重新跑测试用例，询问反馈——只是中间没有浏览器审阅者。如果有文件系统，可以在文件系统上把结果组织成迭代目录。

**描述优化**：本节需要 `claude` CLI 工具（具体是 `claude -p`），只有在 Claude Code 里才有。如果在 Claude.ai，跳过它。

**盲对比**：需要子代理。跳过它。

**打包**：`package_skill.py` 脚本在任何有 Python 和文件系统的地方都可以跑。在 Claude.ai 上可以跑它，用户可以下载生成的 `.skill` 文件。

**更新现有 skill**：用户可能要求你更新现有 skill，而不是创建新的。这种情况下：
- **保留原始名称。** 注意 skill 的目录名和 `name` frontmatter 字段——原样使用不要变。比如如果已安装的 skill 是 `research-helper`，输出 `research-helper.skill`（不是 `research-helper-v2`）。
- **在编辑前复制到可写位置。** 已安装的 skill 路径可能是只读的。复制到 `/tmp/skill-name/`，在那编辑，然后从副本打包。
