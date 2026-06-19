---
name: workflow-commit-brain
description: 简化版 commit skill，自动执行提交。当用户说"/commit"、"帮我提交"、"提交代码"时触发。
---

# Workflow Commit Brain

简化 commit 工作流，自动执行提交。

**适用范围**：仅限 `brain` 仓库。其他仓库请手动操作。

## 触发

- `/commit`
- "帮我提交"
- "提交代码"

## 流程

### 1. 检查当前仓库

确认在 `brain` 目录内执行。

### 2. 自动执行 git status

展示当前状态。

### 3. 检查远程更新

```bash
git fetch origin
git status
```

若有远程更新，提示用户是否需要 merge 或 rebase。

### 4. 分析改动，确定 commit

根据改动内容，生成 1 个规范 commit。

### 5. 自动执行

```bash
git add .
git commit -m "<message>"
git push
```

### 6. 展示结果

显示 commit hash、message 和 push 状态。

## Commit message 规范

```
<type>: <简短描述>
```

类型：feat | fix | docs | refactor | chore | ci | test

要求：中文、祈使语气、≤72 字符、不带句号。
