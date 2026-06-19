# graphify-power

用于**最大化发挥 graphify 的作用**。

## 核心思路

- **先用 graphify 缩小范围**
- **再少量读取源码确认细节**
- **修改代码后更新图谱**
- **高价值问答可写入 memory**

## 优先使用的命令

- `graphify query "问题"`
- `graphify path "A" "B"`
- `graphify explain "X"`
- `graphify affected "X"`
- `graphify update .`
- `graphify save-result ...`

## 适用场景

- 理解架构
- 分析调用链
- 看模块关系
- 评估改动影响
- 沉淀项目知识

## 技能文件

- `./graphify-power/SKILL.md`
