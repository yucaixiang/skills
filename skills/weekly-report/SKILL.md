---
name: weekly-report
description: AI 自动写周报 Skill — 端到端串联"拉数据 → 套模板 → 同步飞书"全流程。用户说"帮我写本周周报"，AI 自动从飞书任务、Git 提交、Jira 拉本周完成项，按公司模板生成结构化周报，自动发送到指定群。这是 SkillForge 的杀手锏组合场景之一。
version: 1.0.0
origin: SkillForge
---

# weekly-report · AI 自动写周报

> 让"周五下午 1 小时写周报"变成"说一句话 = 周报生成 + 发群"。

---

## When to Activate

- 用户说"帮我写周报"、"本周周报"、"周五了"
- 周五下午 4-6 点（自动提示）
- 用户说"上传周报"、"同步给老板"

---

## 工作流：5 步生成周报

```
Step 1 · 拉数据             从飞书任务 / Git / Jira 拉本周完成项
   ↓
Step 2 · 套公司模板         背景 / 进展 / 阻碍 / 下周计划 4 段
   ↓
Step 3 · 智能总结           AI 提炼关键事项，不堆流水账
   ↓
Step 4 · 用户预览修改       展示给用户确认
   ↓
Step 5 · 发送 / 同步        发飞书群 + 写入飞书文档
```

---

## Step 1 · 拉本周完成项

### 数据源

| 来源 | 拉取方式 | Skill |
|---|---|---|
| 飞书任务 | `lark-cli task list --status done --since monday` | `lark-cli/lark-task` |
| Git 提交 | `git log --author=<me> --since="7 days ago" --pretty=format:'%s'` | shell |
| Jira | `jira search --jql "assignee=currentUser() AND status=Done AND updated>=-7d"` | shell / Jira API |
| 飞书云文档贡献 | `lark-cli docs my-recent` | `lark-cli/lark-doc` |
| 飞书 OKR 进展 | `lark-cli okr my-progress` | `lark-cli/lark-okr` |

### 拉取示例

```bash
# 本周已完成飞书任务
lark-cli task list \
  --status done \
  --since "$(date -d 'last monday' +%Y-%m-%d)" \
  --assignee me

# 本周 Git 提交
git log --author="$(git config user.email)" \
  --since="last monday" \
  --pretty=format:'%h %s' | head -20

# 本周飞书文档贡献
lark-cli docs list-mine --since "last monday"
```

---

## Step 2 · 公司周报模板

```markdown
# {Name} 周报 · {2026 W22}（5.30 - 6.5）

## 1. 本周完成

### 1.1 项目进展
- ✅ [项目 A] 完成 xxx，效果 xxx
- ✅ [项目 B] 完成 xxx，效果 xxx
- 🟡 [项目 C] 进行中，预计下周完成

### 1.2 关键产出
- 文档：xxx, xxx
- 代码：xxx PR 已合并
- 评审：参与 xx PRD 评审

## 2. 关键洞察 / 学习

- xxx（一句话）

## 3. 遇到的阻碍

- xxx，需要 @ xxx 协助
- xxx，已计划 xxx 解决

## 4. 下周计划

- 主要目标 1
- 主要目标 2
- 主要目标 3

## 5. 需要老板帮助 / 决策

- xxx（如有）
```

---

## Step 3 · 智能总结的关键

### ✅ 必做

- **从输入数据中找"亮点"，而非堆砌**
  - ❌ "本周完成 PR-123, PR-124, PR-125..."
  - ✅ "完成佣金计算性能优化（PR-123），P99 从 850ms 降到 250ms"

- **量化效果**
  - ❌ "改进了用户体验"
  - ✅ "出金页加载时间 -40%"

- **关联 OKR / 项目目标**
  - 提到"本周完成的事项中，xx 直接推进了 KR-1"

### ❌ 不做

- 不堆流水账
- 不写"加强 / 提高 / 改善"类空话
- 不超过 1 页（A4）

---

## Step 4 · 给用户预览

```
[AI 生成预览]

📝 本周周报已生成，请您 review：

================================
{周报内容}
================================

✏️ 修改建议（如有）：直接说"把 xx 改成 yy"
📤 确认发送：说"发吧"
🗑️ 重新生成：说"重新拉数据"

发送目标：
  - #交易组日报（你的部门日报群）
  - 老板 @ZhangManager（直接 IM）
  - 飞书云文档归档：/我的周报/2026/W22
```

---

## Step 5 · 自动发送

```bash
# 1. 发到部门群
lark-cli im send --chat "#交易组日报" \
  --content "@ZhangManager 我的本周周报：

{周报正文}"

# 2. 写入飞书归档文档
lark-cli docs +create --api-version v2 \
  --content "<title>{name}周报-W22</title>{HTML 化的周报}" \
  --space "/我的周报/2026/W22"

# 3. 同步到 OKR 进展
lark-cli okr update-progress \
  --kr "KR-1" \
  --progress "+15%" \
  --note "完成佣金计算性能优化"
```

---

## 演示场景（SkillForge 杀手锏 1）

### 用户输入

> "帮我写本周周报，发到交易组日报群"

### AI 执行链

```
[15:00] 收到请求

[15:00] 触发 weekly-report Skill
        ↓ 联动 lark-cli/lark-task
[15:00] lark-cli task list --status done --since monday
        → 拉到 8 个已完成任务

[15:00] git log --author=me --since "last monday"
        → 12 个 commit

[15:01] lark-cli docs list-mine --since "last monday"
        → 3 个飞书文档贡献

[15:01] 智能提炼：
        - "佣金计算性能优化"（最大影响）
        - "新代理招募活动 PRD 评审"
        - "线上 P1 故障应急（02:00-02:35）"

[15:01] 套模板生成周报（350 字）

[15:01] 给用户预览：
        [周报内容]
        是否发送？

[用户] 发吧

[15:02] 联动 lark-cli/lark-im
        lark-cli im send --chat "#交易组日报" --content "..."
        ✅ 已发送，群里 17 人将看到

[15:02] 联动 lark-cli/lark-doc
        归档到 /我的周报/2026/W22

完成，全程 2 分钟。
```

### 效果

| 维度 | 之前 | 现在 |
|---|---|---|
| 耗时 | 45 分钟 | 2 分钟 |
| 质量 | 想到啥写啥 | AI 智能提炼 + 量化 |
| 一致性 | 每次格式不同 | 严格遵守公司模板 |
| 归档 | 经常忘 | 自动归档 |

---

## 进阶玩法

### 1 · 月报 / 季报

```
"帮我写本月报"
"帮我写 Q2 季度总结"
```

复用同一套机制，时间范围放大。

### 2 · 团队周报汇总

```
"汇总交易组本周所有周报，生成给老板的简报"
```

AI 拉所有人周报，去重、提炼跨人共性进展。

### 3 · 跨部门简报

```
"基于交易组、运营组、风控组的周报，生成给 CEO 的双周报"
```

---

## 反例

| 反例 | 改正 |
|---|---|
| 全是任务列表，没洞察 | 必须有"关键洞察"段 |
| 全是 PR 标题堆砌 | 必须有"业务影响"说明 |
| 没有下周计划 | 必填 |
| 不提阻碍 | 主动暴露问题更显成熟 |

---

## 与其他 Skill 协作

```
[用户说"写周报"]
       ↓
[本 Skill] 引导流程
       ↓
[lark-cli/lark-task]  拉任务
[lark-cli/lark-doc]   拉文档贡献
[lark-cli/lark-okr]   拉 OKR 进展
       ↓
[本 Skill] 智能提炼
       ↓
[lark-cli/lark-im]    发群
[lark-cli/lark-doc]   归档
```

---

## 引用

- 关联 Skill：lark-cli（核心依赖）, hytech-cpa（如果周报涉及业务数据）
- 公司周报规范（飞书）：/HR/工作汇报规范
