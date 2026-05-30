# 08 · 新员工 1 天上手指南

> 本文档面向第一次接触 hytech-cpa 的同学（研发 / 测试 / 产品 / 运维 / 运营），按"分阶段任务清单"组织。
> 目标：跟着做一天，你就能开始接手第一个小需求或排查第一个生产问题。

---

## 一、上午（理解业务，2-3 小时）

### Step 1 · 知道这是个什么系统（15 分钟）

读 `SKILL.md` 的"项目快速画像"和"4 大业务领域"。

**自测**：你能回答这些问题吗？
- hytech-cpa 是给谁用的？（公司运营 / 代理 / 客户？）
- 替换了哪个系统？
- 4 个业务领域分别叫什么？端口号？
- 多品牌怎么实现？

### Step 2 · 理解核心业务（45 分钟）

读 `docs/02-business-modules.md` 全部 9 大业务模块。

**自测**：
- 给一个真实代理，怎么从账户表查到他名下所有客户？
- 客户从注册到产生第一笔佣金，会经过哪些状态？
- 多层级分润是什么？给个例子。
- BPM 审批走哪几个状态？

### Step 3 · 理解架构（45 分钟）

读 `docs/01-architecture.md` + `CLAUDE.md`。

**自测**：
- ToB 服务能不能直接调 ToC 服务？为什么？
- backend 层为什么不能返回 Entity？
- `/bsn` 和 `/bck` 区别是什么？
- 告警是怎么自动 @ 责任人的？

### Step 4 · 看一遍代码全景（30 分钟）

打开 IDE，按下面顺序点开看一眼（不必看懂细节，先建立"代码在哪"的肌肉记忆）：

```
hytech-cpa/
├── hytech-cpa-admin/cpa-admin-service/cpa-admin-business/
│   └── src/main/java/com/hytech/cpa/admin/business/
│       ├── controller/   ← 这里都是 Bsn*Controller
│       └── service/      ← 这里都是业务 Service
│
├── hytech-cpa-core-backend/src/main/java/com/hytech/cpa/core/backend/
│   ├── business/         ← Core Service（重头戏）
│   │   ├── bonus/
│   │   │   ├── calculate/    ← 佣金计算
│   │   │   ├── share/        ← 分润
│   │   │   └── service/
│   │   └── component/
│   │       └── calculator/   ← 4 个 BonusCalculator 策略实现
│   ├── model/db/         ← 所有 Entity
│   ├── model/dto/        ← 所有 DTO
│   ├── model/convert/    ← MapStruct 转换
│   ├── mapper/           ← Mapper 接口
│   └── repository/       ← Repository 封装
│
├── hytech-cpa-job/cpa-job-executor/src/main/java/com/hytech/cpa/job/executor/
│   ├── bonus/            ← 佣金相关定时任务
│   ├── payment/
│   ├── user/
│   └── transfer/
│
└── hytech-cpa-bpm/src/main/java/com/hytech/cpa/bpm/
    ├── service/          ← ApproveService 入口
    ├── model/            ← Approve* 实体
    └── state/            ← 状态机
```

---

## 二、下午（动手实操，3-4 小时）

### Step 5 · 本地环境搭建（90 分钟）

按 `docs/05-development-guide.md` 第 15 节"本地启动指南"操作：

1. 装好 Java 17、Maven、MySQL、Redis、Consul、Kafka
2. 启动 4 个服务（Admin/Client/Job/Open）
3. 访问各服务的 `/doc.html` 看接口

**Checkpoint**：能成功调通一个 `/bsn/cpa-account/page` 分页接口。

### Step 6 · 读核心业务流程（60 分钟）

读 `docs/06-business-flows.md`，重点流程 1（佣金计算）和流程 2（分润）。

打开下面这 3 个文件对照看：
```
EventHandleCoreServiceImpl.java
BonusCalculationCoreServiceImpl.java
FTDBonusCalculator.java
```

**自测**：
- 一笔入金事件进来，到生成佣金记录，经过哪些 Service？
- 为什么要加 RedissonLock？
- FTD 命中多档为什么不能继续算？

### Step 7 · 跑一遍单测（30 分钟）

```bash
cd hytech-cpa-core-backend
mvn test -Dtest=EventHandleCoreServiceTest
```

看测试用例怎么 mock 数据、怎么断言。

### Step 8 · 修一个小 bug 或加一个小功能（60 分钟）

挑一个真实的入门级 issue（找 mentor 要），跟着做：
1. 在 IDE 里找到对应 Controller
2. 顺着调用链找到 Service / Repository
3. 改代码，写测试
4. 本地验证
5. 提 PR，按 `docs/05-development-guide.md` 第 14 节的 Checklist 自查

---

## 三、晚上（巩固，1 小时）

### Step 9 · 看一遍 FAQ（30 分钟）

读 `docs/07-faq.md`。

**特别注意**：
- 业务概念题（QFTD / ruleSnapshot / path 字段）
- 排查路径题（佣金没生成 / 余额对不上）

### Step 10 · 加入运维群（10 分钟）

- 飞书运维告警群（找 mentor 拉你进）
- 知道告警出现时怎么排查（参考 FAQ Q28-Q30）

### Step 11 · 反思 + 提问（20 分钟）

把你看不懂的地方列出来，明天问 mentor。**强烈推荐用 SkillForge 平台**：

打开 Claude Code，直接问：
- 「hytech-cpa 的佣金计算入口在哪？」
- 「这个 Calculator 类负责什么？」
- 「我想查所有客户的 QFTD 状态，SQL 怎么写？」

AI 会通过 `hytech-cpa` Skill 自动调取本文档系列回答。

---

## 四、第二周目标

### 研发同学

- [ ] 独立完成一个完整需求（含 Controller / Service / Mapper / 单测）
- [ ] 提一个 PR 并通过 Review
- [ ] 能在 mentor 不在时排查一个 P2 级线上问题

### 测试同学

- [ ] 能独立给一个新功能写测试用例
- [ ] 跨岗位求助：用 SkillForge 调用 `hytech-cpa` Skill 自助理解代码逻辑
- [ ] 给 mentor 演示"AI 协助测试用例生成"

### 产品同学

- [ ] 能独立写一份小需求的 PRD（用 `prd-template` Skill）
- [ ] 能用 SkillForge 自助查"历史决策"和"功能现状"
- [ ] 给业务方解释清楚 hytech-cpa 替代 Cellxpert 的价值

### 运维同学

- [ ] 能独立处理一个 P1 告警（参考 `oncall-trading` Skill）
- [ ] 能用 lark-cli 自动建战时群、拉责任人
- [ ] 给 mentor 演示一次"AI 引导的应急处理"

### 运营同学

- [ ] 能用 `data-query` Skill 自助查一个数据需求
- [ ] 能用 `campaign-review` Skill 输出第一份代理活动复盘
- [ ] 不再因为简单 SQL 找 BI

---

## 五、一些"过来人"建议

### 关于业务理解

- **不要试图一次记住所有规则**。佣金规则细节多到能写一本书，等遇到具体问题时再回查 `docs/`
- **画图**。看不懂多层级分润时，自己拿张纸画一遍关系链，比看 100 行代码有用
- **跟一个真实案例**。找一个真实代理，跟踪他从注册到收佣金的全流程

### 关于代码

- **先理解架构，再看实现**。不要一上来就深入某个 Service 看 1000 行实现
- **MDC `eventid` 是你的好朋友**。线上排查时按 eventid 一捞日志就能看到整条链路
- **善用飞书告警**。告警 title 通常已经把根因说清楚

### 关于工具

- **SkillForge + Claude Code 是新员工的最佳搭档**。 装好 `hytech-cpa` Skill 后，AI 几乎能回答所有项目问题
- **Knife4j (`/doc.html`)** 是接口手册，开发自测离不开
- **XxlJob 后台** 是看定时任务执行情况的地方

### 关于沟通

- **跨岗位别绕远路**。测试不必非要 @ 研发，先问 AI（带 `hytech-cpa` Skill）
- **遇到不知道的术语**（如 QFTD），先查 FAQ，查不到再问
- **PR Review 不是审判**，是学习。多看大家给你的 Review 评论

---

## 六、参考资料汇总

| 资料 | 用途 |
|---|---|
| `SKILL.md` | 项目入口 |
| `docs/01-architecture.md` | 架构理解 |
| `docs/02-business-modules.md` | 业务模块 |
| `docs/03-data-model.md` | 数据模型 |
| `docs/04-tech-stack.md` | 技术栈与规范 |
| `docs/05-development-guide.md` | 开发规范 |
| `docs/06-business-flows.md` | 业务流程 |
| `docs/07-faq.md` | 常见问题 |
| `数据迁移doc/hytech-cpa项目理解.md` | 项目原始理解文档（详） |
| README.md | 项目总览 |
| Knife4j `/doc.html` | 接口文档 |
| XxlJob 控制台 | 定时任务管理 |

---

## 七、新人 5 个里程碑

| 里程碑 | 评判标准 |
|---|---|
| 🟢 上手 | 能独立调通一个接口、跑通一个单测 |
| 🟢 入门 | 能独立完成一个小需求并通过 Review |
| 🟢 熟练 | 能独立排查 P2 级线上问题 |
| 🟢 高效 | 能用 SkillForge / Claude Code 把日常工作量降低 30% 以上 |
| 🟢 资深 | 能给业务/产品/测试反向输入，能 Review 别人的设计 |

每达到一个里程碑，找 mentor 打勾。

---

**欢迎加入 hytech-cpa 团队！🎉**
