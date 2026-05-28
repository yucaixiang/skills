# SkillForge 企业知识库

本仓库存储企业所有 Skill 知识文档，由 SkillForge 平台管理和消费。

## 目录结构

```
skills/
├── {skill-name}/
│   ├── SKILL.md          ← 主文档（AI 首先读取）
│   ├── meta.yml          ← 元数据（分类、标签、负责人等）
│   ├── docs/             ← 详细知识文档
│   └── references/       ← 参考资料（代码片段、配置等）
└── _meta/
    └── categories.yml    ← 全局分类定义
```

## 规范

- 每个 Skill 一个独立目录
- SKILL.md 控制在 200-500 行以内
- meta.yml 必填字段：name, displayName, category, description, owner, team, tags
- 变更通过 PR 提交，Skill owner 审核后合并

## 使用方式

- **Web 平台**: SkillForge 管理界面浏览、搜索、编辑
- **Claude Code**: 通过 MCP Server 实时查询
- **本地文件**: clone 本仓库到 ~/.claude/skills/
