---
name: code-deep-reader
description: >
  带我精读一个文件的核心函数（能帮助理解整个项目的"枢纽函数"），主函数逐行分析，被主函数调用的其他函数只给出输入输出示例和功能说明。
  当用户说"精读"、"逐行读"、"主函数"、"入口函数"、"深度分析"某个文件时使用。
  支持中文用户。不生成文件，直接在当前对话中输出。
---

# 代码精读：枢纽函数逐行详解

**核心原则**：
1. **先用 graphify 理解代码关系，再读源码**
2. **每行必读，每行必注**——通过内联注释的方式逐行解释每一行代码
3. **聚焦枢纽函数**，被调函数只给输入输出概览

---

## 触发判断

收到请求后，先确认是否属于"精读主函数"场景：

- 用户指定了**具体文件路径** → 进入精读流程
- 用户只说了**模块名** → 先定位该模块的入口文件（`__main__.py` 或最顶层的 runner/entry 函数）
- 用户说的是**"项目结构"、"架构"、"模块关系"** → 触发 codebase-reader skill，不触发此 skill

---

## 工作流程

### 第一步：graphify 定位枢纽函数

**先 graphify，后读源码。** 用 graphify 快速确定文件中的关键函数：

```bash
# 查询目标文件/模块的函数结构和调用关系
graphify query "<文件名或模块名> 函数结构和调用关系"

# 深入了解某个核心函数
graphify explain "<函数名>"
```

枢纽函数通常具备以下特征：
- **调用其他函数多**（扇出多）
- **被外部模块调用**（扇入多）
- **是项目主流程的关键节点**（如 `run()`、`execute()`、`build()`）

如果 graphify 给出多个候选枢纽函数，列出供用户选择：

```
根据 graphify 分析，这个文件有以下关键函数：

1. `run()` (line 42) — 调用了 f1/f2/f3 等 8 个函数，是最核心的枢纽
2. `_step()` (line 120) — 单次迭代的核心，被 run() 调用
3. `validate()` (line 200) — 验证函数，扇出较少

建议从 `run()` 开始精读，因为它调用了其他所有关键函数。
请告诉我你想精读哪个，或者我默认选 `run()`。
```

---

### 第二步：建立调用关系快照

读取文件后，先建立**调用关系快照**，明确要分析的范围：

```
枢纽函数 `run()` 调用了以下函数：
├── [外部] self.llm.call()          → LLM 客户端调用
├── [外部] self.executor.search()    → 搜索引擎
├── [内部] self._dispatch()          → 工具分发（第 45 行）
└── [内部] self._commit()            → 提交结果（第 78 行）

处理原则：
- [外部] 函数：只说明归属模块和作用，不展开内部实现
- [内部] 函数：需要逐行精读
- 嵌套 ≥3 层：仅列出，不展开
```

---

### 第三步：枢纽函数逐行精读（核心）

**这是最重要的环节，每一行都要有详细的内联注释。**

#### 输出结构

```markdown
## `run()` 函数精读 — 第 42-120 行
```

分段输出，每段 5-20 行，直接在代码下方用编号列表逐行解释：

---

### 第 46-58 行：参数校验与初始化

**代码与逐行注释**：

```46:58:src/example.py
def run(self, mode: str, config: dict) -> dict:
    # ═══════════════════════════════════════════════════════════
    # 分段主题：参数校验与初始化
    # ═══════════════════════════════════════════════════════════

    # ─── 校验运行模式 ───
    # mode 必须是非空字符串，且值必须是已知的模式之一
    # 这里用 set 做 O(1) 查找，比多个 if-elif 更简洁
    if not mode or mode not in {"cold_start", "revise", "resume"}:
        # 非法模式直接抛异常，避免后续无效执行
        raise ValueError(f"Invalid mode: {mode}. Must be one of cold_start/revise/resume")

    # ─── 超时控制初始化 ───
    # config.get() 的第二个参数是默认值，防止 KeyError
    # _TIMEOUT 默认 300 秒，给 LLM 调用留足时间
    timeout = config.get("timeout", 300)  # seconds

    # ─── 状态机初始化 ───
    # self.state 是实例属性，跨调用保持状态
    # 第一次调用时 state 为 None，需要初始化
    if not hasattr(self, "state") or self.state is None:
        self.state = StateMachine()  # 新建状态机，记录当前处于哪个阶段
```

**逐行解释**：

1. **`def run(self, mode: str, config: dict) -> dict:`**
   定义 `run()` 方法，`self` 是实例引用，`mode` 决定运行模式（cold_start/revise），`config` 是配置字典，返回 `dict` 类型。

2. **`# ═══════════════════════════════════════════════════════════`**
   装饰性分隔线，用等号组成，用于视觉区分不同逻辑段。

3. **`# 分段主题：参数校验与初始化`**
   标注本段代码的核心主题，让读者快速了解这段做什么。

4. **`# ─── 校验运行模式 ───`**
   次级分隔线，用短横线组成，标注子逻辑块。

5. **`# mode 必须是非空字符串...`**
   解释为什么需要这个检查：确保调用方传入了合法参数。

6. **`# 这里用 set 做 O(1) 查找...`**
   解释算法选择：set 的 `in` 操作是常数时间，比 list 的线性查找更快。

7. **`if not mode or mode not in {"cold_start", "revise", "resume"}:`**
   条件判断：`not mode` 检查空值/None，`mode not in set` 检查值是否在已知集合中。

8. **`raise ValueError(f"Invalid mode: {mode}...")`**
   非法参数时抛 `ValueError`，用 f-string 动态插入实际传入的 mode 值，便于调试。

9. **`# ─── 超时控制初始化 ───`**
   标注下一子逻辑块。

10. **`# config.get() 的第二个参数是默认值...`**
    解释 `.get()` 方法的用法：安全获取字典值，不抛 KeyError。

11. **`timeout = config.get("timeout", 300)`**
    从 config 中取 timeout，默认 300 秒；结果绑定到 `timeout` 变量。

12. **`# seconds`**
    单位注释，让读者明确 timeout 的单位是秒，不是毫秒。

13. **`# ─── 状态机初始化 ───`**
    标注第三个子逻辑块。

14. **`# self.state 是实例属性，跨调用保持状态`**
    解释 `self.state` 的生命周期：跨多次 `run()` 调用保持。

15. **`# 第一次调用时 state 为 None，需要初始化`**
    说明为什么需要检查 None：初始状态。

16. **`if not hasattr(self, "state") or self.state is None:`**
    复合条件：属性不存在 OR 属性值为 None 时都需要初始化。

17. **`self.state = StateMachine()`**
    创建新的状态机实例，赋值给实例属性。

18. **`# 新建状态机实例，记录当前处于哪个阶段`**
    解释这条赋值语句的业务含义。

**关键决策解析**：

```
为什么用 set 而非 list 做成员检查？
→ set 的 __contains__ 是 O(1)，list 是 O(n)
→ 当模式列表很长时，set 更高效
→ Python 习惯用 set 表示"离散选项集合"

为什么异常用 raise 而非 return error dict？
→ 异常表示"不应该发生的错误"，调用者必须处理
→ return error dict 表示"可预期的失败"，调用者可忽略
→ 这里 mode 参数是程序员接口，错误是调用方的 bug，用异常更合适
```

---

### 第 59-75 行：主循环结构

**代码与逐行注释**：

```59:75:src/example.py
    # ─── 主循环：最多重试 max_retries 次 ───
    # 注意：这是一个"保护性循环"，防止 LLM 无限重试
    # 即使每次都失败，也要有个终止条件
    for attempt in range(max_retries):
        try:
            # ─── 构造请求 ───
            # messages 是对话历史，第一条是 system prompt
            # append() 返回 None，所以不能链式调用
            messages = [self._build_system_prompt()]
            messages.append(self._build_user_prompt(mode, config))

            # ─── 调用 LLM ───
            # timeout 参数传递到底层 HTTP 客户端
            # response 可能是成功结果或异常
            response = self.llm.call(
                messages=messages,
                model=config.get("model", "gpt-4"),
                timeout=timeout,
            )
```

**逐行解释**：

1. **`# ─── 主循环：最多重试 max_retries 次 ───`**
   标注本段主题：主循环，带重试次数限制。

2. **`# 注意：这是一个"保护性循环"...`**
   解释设计意图：防止无限重试，保护系统稳定性。

3. **`# 即使每次都失败，也要有个终止条件`**
   强调：任何循环都应有退出条件，否则可能死循环。

4. **`for attempt in range(max_retries):`**
   `range(max_retries)` 生成 0 到 max_retries-1 的序列，`attempt` 是当前迭代计数器。

5. **`try:`**
   开始异常捕获块，后续代码可能抛出需要捕获的异常。

6. **`# ─── 构造请求 ───`**
   标注子逻辑块：构建发送给 LLM 的请求。

7. **`# messages 是对话历史，第一条是 system prompt`**
   解释 `messages` 的数据结构：列表，第一项是系统提示。

8. **`# append() 返回 None，所以不能链式调用`**
   提醒：`list.append()` 是 in-place 操作，返回 None，不能 `list.append().append()`。

9. **`messages = [self._build_system_prompt()]`**
   创建消息列表，第一项是系统提示；方括号内立即求值调用 `_build_system_prompt()`。

10. **`messages.append(self._build_user_prompt(mode, config))`**
    追加用户提示到消息列表，传入 mode 和 config 参数。

11. **`# ─── 调用 LLM ───`**
    标注下一个子逻辑块。

12. **`# timeout 参数传递到底层 HTTP 客户端`**
    说明 timeout 的去向：传递给网络请求层。

13. **`response = self.llm.call(`**
    调用 LLM，返回值赋给 `response`；括号开始，换行分隔参数。

14. **`messages=messages,`**
    关键字参数：传入对话历史列表。

15. **`model=config.get("model", "gpt-4"),`**
    关键字参数：从 config 取模型名称，默认 "gpt-4"。

16. **`timeout=timeout,`**
    关键字参数：传入超时时间。

17. **`)`**
    括号闭合，调用 `self.llm.call()` 完成。

**语法细节解析**：

```
Q: messages = [self._build_system_prompt()]
   这里 [] 里的表达式什么时候求值？

A: 即时求值（eager evaluation）
   Python 会在创建列表时立即调用 _build_system_prompt()
   不会延迟到后续访问时才调用

Q: 为什么用 messages.append() 而不是 messages + [msg]？

A: append() 是 in-place 操作，返回 None
   messages + [msg] 是新列表，返回新对象
   性能上 append() 更优（无需创建新列表）
```

---

### 第 76-95 行：错误处理与重试逻辑

**代码与逐行注释**：

```76:95:src/example.py
        except LLMError as e:
            # ─── LLM 调用失败 ───
            # LLMError 是自定义异常，表示 API 层面的错误（如网络超时、API 限流）
            # 注意：不是所有异常都重试——业务逻辑错误不应该重试

            if attempt == max_retries - 1:
                # 最后一次尝试也失败了，无法恢复
                # 用 return 而非 raise，保持函数签名的"返回型"风格
                return {
                    "ok": False,
                    "error": f"LLM call failed after {max_retries} attempts: {e}",
                    "attempts": attempt + 1,
                }

            # ─── 重试前的冷却 ───
            # 指数退避（exponential backoff）：每次失败后等待时间翻倍
            # 公式：wait = base * (2 ** attempt)
            # attempt=0: wait=1s, attempt=1: wait=2s, attempt=2: wait=4s
            wait_time = base_wait * (2 ** attempt)
            time.sleep(wait_time)  # 阻塞等待，不消耗 CPU

            # 记录重试日志，但不中断循环
            if self.logger:
                self.logger.warning(f"Attempt {attempt+1} failed, retrying in {wait_time}s: {e}")

            continue  # 跳到下一次循环，继续重试
```

**逐行解释**：

1. **`except LLMError as e:`**
   捕获 `LLMError` 类型异常（自定义异常），绑定到变量 `e` 供后续使用。

2. **`# ─── LLM 调用失败 ───`**
   标注子逻辑块主题。

3. **`# LLMError 是自定义异常...`**
   解释异常来源：不是 Python 内置异常，是项目自定义的 API 层异常。

4. **`# 注意：不是所有异常都重试——业务逻辑错误不应该重试`**
   强调重试策略：只重试可恢复的网络/服务错误，业务错误不应重试。

5. **`if attempt == max_retries - 1:`**
   检查是否是最后一次尝试：`max_retries - 1` 是最大索引值（从 0 开始）。

6. **`# 最后一次尝试也失败了，无法恢复`**
   解释为什么需要这个特殊处理：已达重试上限，没有更多机会了。

7. **`# 用 return 而非 raise，保持函数签名的"返回型"风格`**
   解释返回风格选择：用返回值而非异常来报告失败。

8. **`return {`**
   返回错误结果字典，键 `"ok": False` 表示失败。

9. **`"error": f"LLM call failed after {max_retries} attempts: {e}",`**
   错误消息包含重试次数和异常信息，便于调试。

10. **`"attempts": attempt + 1,`**
    记录本次共尝试了多少次（+1 因为 attempt 从 0 开始）。

11. **`# ─── 重试前的冷却 ───`**
    标注下一个子逻辑块。

12. **`# 指数退避（exponential backoff）...`**
    解释算法名称和用途：避免频繁重试触发限流。

13. **`# 公式：wait = base * (2 ** attempt)`**
    用注释展示公式，让读者能手动验证计算结果。

14. **`wait_time = base_wait * (2 ** attempt)`**
    `**` 是幂运算符（不是 XOR），计算等待时间。

15. **`time.sleep(wait_time)`**
    线程阻塞 `wait_time` 秒，期间不消耗 CPU，但会阻塞当前线程。

16. **`# 记录重试日志，但不中断循环`**
    说明日志记录不影响控制流：只是记录，不改变执行顺序。

17. **`if self.logger:`**
    检查 logger 是否存在，避免对 None 调用方法。

18. **`self.logger.warning(f"Attempt {attempt+1} failed...")`**
    记录警告级别日志，包含重试次数、等待时间和异常信息。

19. **`continue`**
    跳过本次循环剩余代码，直接进入下一次迭代。

**设计意图**：

```
为什么用指数退避而不是固定等待？
→ API 限流通常有"时间窗口"，固定等待可能恰好落在同一窗口
→ 指数增长让请求分布更均匀，减少再次触发限流的概率

为什么用 continue 而不是 break？
→ continue 继续循环，尝试下一次
→ break 会退出循环，无法重试
→ 这里语义是"失败了但还要试"，所以 continue
```

---

### 第 96-110 行：成功返回路径

**代码与逐行注释**：

```96:110:src/example.py
        # ─── 正常返回路径 ───
        # 能执行到这里说明 LLM 调用成功，没有异常
        # 但这不保证 LLM 返回了有效结果，还需要检查 response

        if not response or not response.content:
            # ─── LLM 返回空内容 ───
            # 这不是异常，是 LLM 的正常行为（有时会拒绝回答或说不理解）
            # 按业务逻辑，这种情况下应该视为失败，让调用方决定如何处理
            return {
                "ok": False,
                "error": "LLM returned empty response",
                "response": response,  # 保留原始响应，便于调试
            }

        # ─── 解析并返回 ───
        return {
            "ok": True,
            "data": response.content,
            "attempts": attempt + 1,  # 记录用了多少次尝试
            "latency_ms": response.latency,  # 性能指标
        }
```

**逐行解释**：

1. **`# ─── 正常返回路径 ───`**
   标注子逻辑块主题：成功执行后的返回处理。

2. **`# 能执行到这里说明 LLM 调用成功，没有异常`**
   说明执行到这里意味着什么：LLM 网络调用正常完成。

3. **`# 但这不保证 LLM 返回了有效结果，还需要检查 response`**
   强调：调用成功 ≠ 结果有效，LLM 可能返回空内容或拒绝回答。

4. **`if not response or not response.content:`**
   双重检查：响应对象存在 AND 响应内容非空。

5. **`# ─── LLM 返回空内容 ───`**
   标注子分支主题。

6. **`# 这不是异常，是 LLM 的正常行为`**
   区分异常类型：这不是网络/服务端错误，是 LLM 正常但无内容的响应。

7. **`return { "ok": False, "error": "LLM returned empty response", ... }`**
   返回失败状态，错误信息说明 LLM 未提供有效内容。

8. **`"response": response,  # 保留原始响应，便于调试`**
   把原始响应也返回，方便调用方查看 LLM 实际说了什么。

9. **`# ─── 解析并返回 ───`**
   标注成功返回路径。

10. **`"ok": True,`**
    成功标志：`True`。

11. **`"data": response.content,`**
    把 LLM 返回的内容放在 `data` 字段。

12. **`"attempts": attempt + 1,  # 记录用了多少次尝试`**
    记录本次执行尝试了多少次，用于评估效率。

13. **`"latency_ms": response.latency,  # 性能指标`**
    记录耗时，供监控和分析使用。

---

### 第四步：被调函数速览

对枢纽函数调用的每个**同文件函数**，输出以下信息（**不展开内部实现**）：

```markdown
### `_build_system_prompt()` — 第 200-215 行

**功能**：构建系统提示词，包含角色设定、能力边界、输出格式要求。

**为什么这样设计**：
- 系统提示决定 LLM 的"人格"，是 prompt engineering 的核心
- 集中在一个函数里，便于维护和修改
- 支持模板变量注入，实现动态内容

**调用链**：
- 被 `run()` 第 68 行调用
- 内部调用 `self._load_template()` 加载模板文件

**输入示例**：
```python
_build_system_prompt()
# Returns: "你是一个专业的 Python 开发者，擅长重构和分析代码。..."

_build_system_prompt(extra_rules={"no_monkey_patching": True})
# Returns: 包含额外规则的扩展系统提示
```

**输出示例**：
```python
_build_system_prompt()
# Returns: str — 系统提示词文本

_build_system_prompt() when template not found
# Raises: FileNotFoundError — 模板文件缺失
```
```

---

### 第五步：总结

```markdown
## `run()` 函数总结

**职责**：作为入口函数，驱动 LLM 交互循环，处理重试逻辑，返回结构化结果。

**输入**：
- `mode`: 字符串，'cold_start' | 'revise' | 'resume'
- `config`: 字典，包含 timeout、model、max_retries 等配置

**输出**：
```python
{
    "ok": bool,           # 是否成功
    "data": Any | None,   # 成功时返回的数据
    "error": str | None,  # 失败时的错误信息
    "attempts": int,      # 尝试次数
    "latency_ms": float   # 耗时（仅成功时）
}
```

**关键特点**：

1. **重试保护**：用指数退避策略，最多重试 max_retries 次
2. **错误分离**：
   - `LLMError` → 重试后最终返回错误字典
   - 空响应 → 直接返回错误字典，不重试
   - 其他异常 → 向上传播（未捕获）
3. **状态保持**：`self.state` 跨调用保持，支持断点续传

**代码风格亮点**：
- 返回字典统一结构，调用方无需 try-except
- 注释密度高，每行都有解释
- 关键决策处有 Q&A 解析

---

**需要进一步精读吗？**
- `_dispatch()` 的工具分发逻辑（行 250-280）比较复杂，要深入看吗？
- 或者想了解 `StateMachine` 状态机的实现？
```

---

## 输出格式规范

1. **标题层级**：`##` 函数名 / `###` 分段标题
2. **代码引用**：使用 `startLine:endLine:filepath` 格式
3. **逐行注释**：在代码下方用编号列表，每条解释一行代码
4. **关键决策解析**：用代码块展示，不混在逐行解释里
5. **语法细节 Q&A**：针对可能困惑的语法点做问答式解释
6. **分隔线**：用分隔线隔开不同逻辑段
7. **函数太长（>80行）**：拆成多个分段，每段独立输出，每段结尾问"继续下一段？"

---

## 触发词

用户可能说：
- "精读这个文件"
- "逐行分析"
- "帮我理解 run() 函数"
- "/code-deep-reader" 后跟文件名
- "带我读一下 main"
- "这个枢纽函数做了什么"
- "帮我深度分析 xxx.py"
- "每一行都给我解释一下"

---

## 边界情况

- **文件没有枢纽函数**（纯数据/常量文件）：告知用户这不是代码文件，询问是否要读其他文件。
- **函数太长（>150行）**：拆成多个逻辑段（每 50 行一段），分批输出，每段结尾问"继续下一段？"
- **函数调用 10+ 个函数**：只精读同文件的核心函数（3-5个），外部调用列出即可
- **graphify 没有目标文件的数据**：直接读取文件，用代码扫描替代——找"调用其他函数最多的函数"，按扇出数排序确定枢纽
- **异步/生成器函数**：注意 `async def` 和 `yield` 的特殊性，说明协程/迭代器如何被驱动
- **用户指定了具体函数名**：直接精读该函数，用 graphify 确认其在调用链中的位置
