---
name: hr-onboarding
description: 新员工入职流程 Skill — HR 自动完成新人入职的全套动作：建飞书账号、拉相关业务群、安排 mentor、发欢迎语、推送入职指南。与 lark-cli 深度联动，将 HR 人均 4 小时/人的入职接待降到 15 分钟。当 HR 说"xx 入职"时调用。
version: 1.0.0
origin: SkillForge
---

# hr-onboarding · 新员工入职流程

> HR 自助 + AI 自动化，覆盖入职第 1 天的所有标准动作。

---

## When to Activate

- HR 说"xx 同学今天入职"、"安排 xx 入职"
- 周一批量入职潮（每周固定时间）

---

## 标准入职 8 步

```
1. 创建飞书账号（HRBP 完成）
2. 自动加入对应业务群（本 Skill + lark-cli）
3. 自动发送欢迎语（本 Skill）
4. 推送入职指南（本 Skill）
5. 安排 mentor 对接（本 Skill）
6. 发放设备清单 & 权限申请（lark-cli/lark-task）
7. 第 1 周培训计划（lark-cli/lark-calendar）
8. 入职评估调度（30 / 60 / 90 天）
```

---

## 自动化执行示例

### 用户输入

> "今天有 3 个新同学入职：张三（hytech-cpa 后端研发）、李四（QA）、王五（产品）"

### AI 执行链

```bash
# 1. 拉群（按岗位匹配）
lark-cli im add-members --chat "#hytech-cpa-dev" --users "@张三"
lark-cli im add-members --chat "#hytech-cpa-qa"  --users "@李四"
lark-cli im add-members --chat "#hytech-cpa-pm"  --users "@王五"

# 2. 通用群（全员通知）
lark-cli im add-members --chat "#公司全员" --users "@张三 @李四 @王五"

# 3. 发欢迎语（个人卡片）
for u in 张三 李四 王五; do
  lark-cli im send --chat "$u" --content "$(welcome_template $u)"
done

# 4. 推送入职指南文档
lark-cli docs share \
  --doc "https://x.larksuite.com/wiki/入职指南" \
  --users "@张三 @李四 @王五"

# 5. 安排 mentor 1:1（用 lark-calendar）
lark-cli calendar create-event \
  --title "新人 1:1 - 张三 & 李 Mentor" \
  --time "2026-06-01 14:00" --duration 30 \
  --attendees "@张三 @李Mentor"

# 6. 创建入职 task 清单
lark-cli task create --title "张三入职 Checklist" \
  --subtasks "签合同,办工牌,装电脑,接入 VPN,加飞书群,认识团队"
```

---

## 欢迎语模板

```
你好 @{name}，欢迎加入！🎉

入职第一天请关注：

📚 入职必读：[飞书链接]
🛠️ 设备领取：找 IT 同学 @ITSupport
👥 你的 Mentor：@{mentor}，今天下午 2 点会和你 1:1
🌐 加入业务群：已自动拉你进 {teamGroup}

入职第一周建议：
- 第 1 天：环境搭建 + 读 README
- 第 2-3 天：跑通本地 + 看 SkillForge 平台 hytech-cpa Skill
- 第 4-5 天：跟着 Mentor 改一个小需求

有任何问题 @ HR-Bot 自助查询，复杂问题 @ {hr-name}。

祝你工作愉快！
```

---

## 按岗位的入职推荐 Skill

| 岗位 | 必装 Skill | 用途 |
|---|---|---|
| 后端研发 | hytech-cpa / dev-standards / code-review-rules / code-walkthrough | 上手项目 + 规范对齐 |
| 测试 | hytech-cpa / test-case-generator / api-test-automation / code-walkthrough | 设计 + 跑用例 |
| 产品 | prd-template / feasibility-checker / competitor-research / hytech-cpa | 写 PRD + 评估可行性 |
| 运维 | oncall-trading / incident-response / risk-control | 值班 + 应急 |
| 运营 | data-query / campaign-review / user-analysis | 数据自助 + 复盘 |
| HR | hr-onboarding（本 Skill） / lark-cli | 入职自动化 |

新人入职第 1 天，自动 push 一条飞书消息：

```
你的角色是 {岗位}，建议安装以下 Skill 到本地 Claude Code：

bash:
  curl -s "http://skillforge:8081/api/skills/name/hytech-cpa/install" | sh
  curl -s "http://skillforge:8081/api/skills/name/dev-standards/install" | sh
  ...
```

---

## 30 / 60 / 90 天评估

| 节点 | 评估内容 | 工具 |
|---|---|---|
| 30 天 | 转正预评估 | 飞书表单 + lark-cli/lark-task |
| 60 天 | 业务掌握度评估 | mentor 反馈 + 业务输出 |
| 90 天 | 正式转正答辩 | 飞书多维表格记录 |

每次自动建会议 + 提醒 + 收集表单。

---

## 引用

- 关联 Skill：lark-cli, hytech-cpa（按岗位）
- HR 知识库（飞书）：/HR/入职流程
