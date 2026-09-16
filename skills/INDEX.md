# 项目 Skills 索引

项目级 Skill 统一放在 `skills/<skill-name>/SKILL.md`。

处理任务时先根据本索引判断是否有匹配的 Skill，只加载当前任务需要的 `SKILL.md`。

| Skill | 适用场景 | 入口 |
|---|---|---|
| `canvas-mindmap` | 创建或整理主题式 Obsidian Canvas 思维导图 | [`canvas-mindmap/SKILL.md`](canvas-mindmap/SKILL.md) |

## 维护约定

- 新增、重命名或删除 Skill 时，只需同步更新本索引。
- Skill 名称、目录名和 `SKILL.md` frontmatter 中的 `name` 保持一致。
- 每个 Skill 的描述应明确适用场景，避免代理加载无关说明。

