# Brain

Agent 的记忆库，存放 skills 和学习沉淀。

## Skills

放在 `brain/skills/` 下（实际路径）。

### 创建新 Skill

```bash
mkdir -p brain/skills/<category>/<skill-name>
# 创建 SKILL.md
# 注册到 brain/skills/index.json
```

### 示例：创建 code-learn

```bash
mkdir -p brain/skills/coding/code-learn
# 写 SKILL.md
# 注册到 index.json
```

### Skill 触发词

| Skill | 触发词 |
|-------|--------|
| workflow-commit-brain | `/commit`、`帮我提交` |
| code-learn | `/code-learn`、`消化这段修改` |

## Learnings

`brain/learnings/` - 代码修改的学习沉淀，按日期存储。

## Guides

`brain/guide/` - 基础设施参考指南（如 ceph 挂载）。

`brain/skills/` 是实际目录，直接创建/编辑即可。
