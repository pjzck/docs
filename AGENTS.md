# AGENTS.md

This file provides guidance to AI coding agents when working with files in this repository.

## 仓库定位

pjzck 组织文档仓库，记录设计方案、技术知识、经验总结、问题排查等内容。

## 目录结构

- `design/` — 设计方案与技术选型
- `knowledge/` — 系统性技术知识
- `experience/` — 通用经验总结
- `troubleshooting/` — 问题排查复盘
- `routemap/` — 流程路线图、思路/伪代码与思维导图（含 Obsidian Canvas）
- `skills/` — 项目级可复用工作流，每个 Skill 位于独立目录的 `SKILL.md`

## 项目级 Skills

- 项目级 Skill 的清单和适用场景统一维护在 `skills/INDEX.md`。
- 处理任务前，先查看索引；存在匹配项时完整读取对应的 `SKILL.md` 并遵循其中约定。
- 只加载当前任务需要的 Skill，不要一次性读取整个目录。
- 新增、重命名或删除 Skill 时更新索引，无需修改本文件。

## 文档命名规范

- 文件名使用小写英文 + 连字符，如 `api-design_ai.md`
- 人工撰写的文档以 `<topic>.md` 命名
- AI 辅助产出的文档以 `<topic>_ai.md` 命名
- 思维导图以核心主题命名为 `<主题>.canvas`，统一直接放在 `routemap/` 下，不创建子目录

## 内容格式

- 标题层级清晰，从 `#` 开始
- 代码块标注语言
- 图片使用相对路径
