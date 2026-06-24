# evals.json Schema

用于 `evals/evals.json` 中测试用例的结构定义。

```json
{
  "skill_name": "manual",
  "evals": [
    {
      "id": 1,
      "eval_name": "descriptive-kebab-name",
      "prompt": "用户会实际说的一句话任务",
      "expected_output": "对最终产物的可验证描述",
      "files": [],
      "assertions": []
    }
  ]
}
```

## 字段说明

- `id`: 整数，测试用例唯一标识。
- `eval_name`: kebab-case 名称，描述这个 eval 在测什么。
- `prompt`: 真实用户 prompt（不是技能描述）。
- `expected_output`: 可验证的预期结果描述，用于人类审阅和后续断言。
- `files`: 输入文件路径列表（本次 skill 不需要外部输入文件，保持 `[]`）。
- `assertions`: 断言数组，见下方。初期留 `[]`，后续迭代补。

## 断言格式

```json
{
  "id": 1,
  "description": "一句话描述这个断言在检查什么",
  "type": "file_exists | contains | not_contains | line_count",
  "path": "relative/path/from/skill/or/workspace",
  "expect": "...regex 或子串...",
  "case_sensitive": false
}
```

- `file_exists`: 检查文件是否存在。
- `contains` / `not_contains`: 检查文件内容是否包含/不包含给定子串。
- `line_count`: 检查文件行数范围，`expect` 写成 `"<N"` 或 `">N"` 或 `"==N"` 或 `"BETWEEN N M"`。

断言数组最终由 `agents/grader.md` 评分器读取，字段必须保持 `text`、`passed`、`evidence` 三件套。
