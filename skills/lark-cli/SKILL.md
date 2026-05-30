---
name: lark-cli
description: 飞书官方 CLI 工具集成 Skill — 让 AI 在终端直接操作飞书消息、文档、多维表格、电子表格、幻灯片、日历、邮箱、任务、会议、知识库、OKR、审批等。开箱自带 26 个子 Skill。当用户需要发飞书消息 / 建群 / 写飞书文档 / 操作飞书表格 / 安排日程 / 收发邮件 / 总结群讨论 / 写周报发群 / 自动建会议室 / 查 OKR / 审批等任意飞书办公场景时立即调用本 Skill。
version: 1.0.0
origin: SkillForge
---

# lark-cli · 飞书官方 AI Agent CLI 集成

> **一次集成，26 种飞书操作能力开箱即用。**

[`lark-cli`](https://github.com/larksuite/cli) 是飞书官方推出的、为 AI Agent 原生设计的命令行工具，覆盖 18 大业务域、200+ 命令、26 个 Claude Code Skill。SkillForge 把它打包成一个统一的"元 Skill" — 安装一次，AI 就拥有了操作公司飞书办公环境的全部能力。

---

## When to Activate

只要用户的请求涉及"飞书 / Lark"任意一个产品功能，立即调用本 Skill。常见触发：

| 触发关键词 | 典型场景 | 调用子能力 |
|---|---|---|
| 发消息 / 建群 / 拉人 / 查群历史 / 撤回 | 团队沟通 | `lark-im` |
| 写文档 / 读文档 / 翻译 / 总结 / 改写 | 文档协作 | `lark-doc` `lark-markdown` |
| 读表格 / 写表格 / 导出表格 | 数据处理 | `lark-sheets` `lark-base` |
| 安排会议 / 查日程 / 找会议室 | 日程管理 | `lark-calendar` |
| 收发邮件 / 起草回复 | 邮件沟通 | `lark-mail` |
| 建任务 / 查任务 / 完成任务 | 任务管理 | `lark-task` |
| 查会议记录 / 听会议纪要 | 视频会议 | `lark-vc` `lark-minutes` |
| 写 OKR / 查 OKR / 对齐 OKR | 目标管理 | `lark-okr` |
| 审批 / 同意 / 拒绝 / 抄送 | 审批流 | `lark-approval` |
| 查考勤打卡 | 考勤 | `lark-attendance` |
| 上传下载文件 / 操作云空间 | 云空间 | `lark-drive` |
| 创建管理幻灯片 | 演示 | `lark-slides` |
| 操作画板 SVG | 白板协作 | `lark-whiteboard` |
| 知识库节点 / 文档管理 | Wiki | `lark-wiki` |
| 找人 / 查通讯录 | 通讯录 | `lark-contact` |
| 开发飞书 HTML/Web 应用 | 应用开发 | `lark-apps` |

特殊触发：
- 用户给出飞书 URL（`/docx/` `/wiki/` `/sheets/` `/base/`）→ 直接用本 Skill 读取
- 用户说"按周报模板发飞书群"等组合场景 → 联动其他 Skill + 本 Skill

---

## 完整能力清单（26 个子 Skill）

| # | Skill | 能力 |
|---|---|---|
| 1 | `lark-approval` | 审批：同意/拒绝/转交/撤回/抄送审批实例，查询审批任务 |
| 2 | `lark-apps` | 开发部署 HTML/Web 应用 |
| 3 | `lark-attendance` | 个人考勤打卡记录查询 |
| 4 | `lark-base` | 多维表格：数据表、字段、记录、视图、仪表盘、自动化、表单、角色权限、聚合分析 |
| 5 | `lark-calendar` | 日历：查看/创建/更新日程，邀请参会人、查找会议室、查询忙闲与时间建议 |
| 6 | `lark-contact` | 通讯录：按姓名/邮箱/手机号搜索用户、获取用户信息 |
| 7 | `lark-doc` | 云文档：创建/读取/更新文档、提取/总结/改写/翻译，读写素材与画板 |
| 8 | `lark-drive` | 云空间：上传下载文件、搜索文档与知识库、管理评论 |
| 9 | `lark-event` | 飞书事件回调订阅与处理 |
| 10 | `lark-im` | 即时通讯：收发消息、回复、创建管理群聊、查看聊天记录与话题、搜索消息、下载媒体 |
| 11 | `lark-mail` | 邮箱：浏览/搜索/阅读邮件、发送/回复/转发邮件、管理草稿、监听新邮件 |
| 12 | `lark-markdown` | Markdown：创建/读取/局部 patch/覆盖更新 Drive 中的原生 `.md` 文件 |
| 13 | `lark-minutes` | 视频会议纪要：查询会议记录、产物、录制 |
| 14 | `lark-okr` | OKR：查询/创建/更新 OKR，管理目标、关键结果、对齐、指标、进展记录 |
| 15 | `lark-openapi-explorer` | 飞书开放 API 探索（兜底通用调用） |
| 16 | `lark-shared` | 共享能力：认证、权限处理（其他 Skill 的前置依赖） |
| 17 | `lark-sheets` | 电子表格：创建/读取/写入/追加/查找/导出表格数据 |
| 18 | `lark-skill-maker` | 帮你生成新的飞书 Skill |
| 19 | `lark-slides` | 幻灯片：创建管理演示文稿、读取内容、新增/删除页面 |
| 20 | `lark-task` | 任务：创建/查询/更新/完成任务，管理任务清单、子任务、评论、提醒 |
| 21 | `lark-vc` | 视频会议：搜索会议记录、查询纪要产物与录制 |
| 22 | `lark-vc-agent` | 视频会议 Agent（智能会议助手） |
| 23 | `lark-whiteboard` | 白板：操作画板、SVG 画板 |
| 24 | `lark-wiki` | 知识库：创建管理知识空间、节点、文档 |
| 25 | `lark-workflow-meeting-summary` | 工作流：会议自动总结 |
| 26 | `lark-workflow-standup-report` | 工作流：自动生成站会报告 |

---

## 工作机制

### 第一次使用

SkillForge 在安装本 Skill 时，会通过 `gitRepos` 配置自动克隆 [`lark-cli` 源仓库](https://github.com/larksuite/cli) 到本地 `.claude/skills/lark-cli/repos/cli/`。仓库里包含：
- 完整 `lark-cli` 二进制源码
- 26 个子 Skill 的 `SKILL.md`（位于 `cli/skills/{name}/`）

### AI 调用流程

```
用户："帮我把本周已完成的飞书任务整理成周报发到《交易组日报》群"
   ↓
AI 识别：涉及"飞书任务 + 飞书消息" → 触发 lark-cli Skill
   ↓
AI 调度子 Skill：
   ① 读 .claude/skills/lark-cli/repos/cli/skills/lark-task/SKILL.md
      → 学会怎么用 lark-cli 查询本周完成任务
      → 执行：lark-cli task list --status done --since "2026-W22-mon"
   ② 读 .claude/skills/lark-cli/repos/cli/skills/lark-im/SKILL.md
      → 学会怎么用 lark-cli 发消息到指定群
      → 执行：lark-cli im send --chat "交易组日报" --content "<周报内容>"
   ↓
任务完成，反馈给用户
```

---

## 前置准备（首次使用必读）

> AI 检测到本 Skill 被首次激活时，**必须先执行下面的安装与认证检查**。

### Step 1 · 安装 lark-cli

```bash
# 方式一：npm（推荐）
npx @larksuite/cli@latest install

# 方式二：从源码（需 Go 1.23+）
cd .claude/skills/lark-cli/repos/cli && make install
```

检测是否已安装：
```bash
which lark-cli && lark-cli --version
```

### Step 2 · 配置应用凭证（仅需一次）

```bash
lark-cli config init   # 交互式引导
```

需要管理员在飞书开放平台创建一个应用，拿到 `App ID` 和 `App Secret`。

### Step 3 · 登录授权

```bash
lark-cli auth login --recommend   # --recommend 自动选择常用权限
```

成功后凭证存储在 OS 原生密钥链中，安全可控。

### Step 4 · 验证

```bash
lark-cli calendar +agenda           # 看今天日程
lark-cli im send --chat "我自己" --content "Hello from AI"   # 给自己发消息测试
```

---

## 与企业上下文的深度联动（SkillForge 增强）

仅有 `lark-cli` 是"通用飞书操作"。SkillForge 通过**注入企业上下文**，把通用能力升级为"懂你公司"的能力：

| `lark-cli` 原始能力 | + 企业上下文 = SkillForge 增强 |
|---|---|
| `lark-im` 发消息 | 用户说"发给我们交易组" → 自动注入 `oc_xxx 交易组群 ID` |
| `lark-doc` 写文档 | 用户说"同步到产品组空间" → 自动定位飞书空间路径 |
| `lark-calendar` 建会议 | 用户说"拉上 trading-engine 相关人" → 从 `hytech-cpa` Skill 读 OWNERS |
| `lark-sheets` 读表格 | 用户说"读本月拉新数据" → 自动定位"2026Q2 拉新"表格 ID |
| `lark-task` 查任务 | 用户说"本周完成的开发任务" → 按当前用户 + 日期范围过滤 |

实现方式：SkillForge 平台维护一份"上下文映射"配置（团队群名 → chat_id、岗位 → OWNERS、模板表格名 → token），AI 在调用 lark-cli 前自动替换。

---

## 杀手锏组合场景

### 场景 1：AI 自动写周报发飞书群

```
用户：「帮我写本周周报，发到交易组日报群」

AI 调用链：
  1. weekly-report skill（公司周报模板，SkillForge 内部）
  2. lark-cli/lark-task（拉本周已完成的飞书任务）
  3. lark-cli/lark-im（发到指定群）

执行：
  ✓ 从飞书任务系统拉本周 8 条已完成任务
  ✓ 按"背景/进展/阻碍/下周计划"模板生成
  ✓ 自动发送到"#交易组日报"群

效果：员工说一句话 = 30 分钟工作量消失
```

### 场景 2：AI 总结群讨论 → 生成会议纪要

```
用户：「把交易组今天的讨论总结成会议纪要发到知识库」

AI 调用链：
  1. lark-cli/lark-im（拉群历史）
  2. lark-cli/lark-doc 或 lark-wiki（写入知识库）

执行：
  ✓ 抓取群消息（按时间窗）
  ✓ 提炼"议题 / 结论 / 待办"三段式
  ✓ 自动建文档 + 归档到指定知识库节点
  ✓ @ 相关人确认
```

### 场景 3：AI 自动安排会议拉相关人

```
用户：「跟 hytech-cpa 的负责人约个会讨论新需求」

AI 调用链：
  1. hytech-cpa skill（读 OWNERS）
  2. lark-cli/lark-calendar（查所有人忙闲，找时间）
  3. lark-cli/lark-calendar（建会，订会议室）
  4. lark-cli/lark-im（发邀请）
```

### 场景 4：AI 收口邮件 → 自动起草回复

```
用户：「给我看新邮件，紧急的帮我起草回复」

AI 调用链：
  1. lark-cli/lark-mail（拉未读邮件，按标签/发件人分类）
  2. lark-cli/lark-mail（起草回复，保留语气模板）
```

---

## 安全提醒

- ✅ 凭证使用 OS 原生密钥链存储，不会泄露到日志/文件
- ✅ 输入防注入，终端输出净化（lark-cli 内置）
- ⚠️ 发送消息 / 修改文档 / 删除内容前，AI 必须**先向用户确认目标对象**（防止误操作）
- ⚠️ 涉及外部域名（如客户邮件）须二次确认
- 🔒 生产环境 `App Secret` 走 AWS Bedrock 合规通道，不出公司

---

## 元信息

| 字段 | 值 |
|---|---|
| 上游来源 | https://github.com/larksuite/cli |
| 维护方 | 飞书官方 |
| 协议 | MIT |
| 子 Skill 数 | 26 |
| 覆盖业务域 | 18 |
| 二进制依赖 | `lark-cli`（自动安装） |
| SkillForge 集成方式 | gitRepos 自动同步 + 元 Skill 调度 |
