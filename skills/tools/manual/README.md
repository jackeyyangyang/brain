# manual

项目"运行手册"维护 skill。

## 触发场景

- 代码改动后需要同步"怎么跑"文档
- 新增/修改模块的 CLI 参数
- 要维护 bash 脚本模板（变量暴露在顶部）
- 要维护最简命令速查（code/）
- 要补坑点/注意事项（guide/）防止 LLM 产生项目幻觉
- 用户说"更新运行文档 / 加个脚本 / 记个坑 / 怎么跑"

## 目录结构

```
manual/
├── bash/<module>.sh      # 每个模块一个，变量在顶部可编辑
├── code/<module>.md      # 每个模块一个，最简 CLI 参数速查
└── guide/<topic>.md      # 按主题，一句话结论 + 坑点 + 相关文件
```

## 快速使用

在 Cursor 里对 skill-enabled 的 agent 说：

- "新增模块 X，帮我补运行手册"
- "render 脚本加了 --samples 参数，更新一下 manual/"
- "把坐标系的坑记到 guide/ 里"
- "整理一下 manual/ 命名，符合 manual 约定"

## 配合其他 skill

- 代码改完后先跑 `graphify update .` 同步知识图谱。
- 如果用户问"怎么跑"或"有什么坑"，直接指向 `manual/` 对应文件。

## 内部文件

- `SKILL.md` — 主 skill 定义。
- `references/schemas.md` — evals.json 字段和断言格式参考。
- `evals/evals.json` — 3 个测试用例，覆盖新模块、参数变更、坑点补录。

## Skill 名称

`manual`（以 `manual` 技能名称触发）。
