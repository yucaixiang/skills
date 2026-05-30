---
name: hytech-cpa
description: hytech-cpa 项目知识库 — 公司自研代理佣金管理系统（CPA, Cost Per Action），用于替换第三方 Cellxpert。面向 Forex/CFD 业务，覆盖代理账户、客户、佣金规则、协议、佣金计算、多层级分润、结算支付、BPM 审批、报告统计全流程。在 hytech-cpa 项目中编码、改代码、排查问题、回答业务问题时必须调用本 Skill。
version: 1.0.0
origin: SkillForge
---

# hytech-cpa · 代理佣金管理系统

公司自研的代理佣金管理系统（CPA, Cost Per Action），替换第三方系统 Cellxpert。
面向 Forex/CFD 在线交易业务，代理人（IB/CPA）邀请客户开户交易，公司按规则向代理支付佣金。

---

## When to Activate

这个 Skill 应该在以下场景被自动调用：

- 在 `hytech-cpa` 项目中编写代码 / 修改代码 / 重构 / Review
- 询问"代理账户、客户、佣金规则、协议、佣金计算、分润、结算、审批"任意业务概念
- 询问"应该改哪个模块 / Controller 在哪 / 这个表对应哪个 Entity"
- 询问"为什么这个佣金没生成 / 这个分润金额不对 / 这个结算单状态为什么是 P"
- 询问"hytech-cpa 的架构 / 分层规则 / 4 个领域是什么"
- 新员工入职、跨岗位（测试/产品/运维）需要快速理解项目

---

## 项目快速画像

| 维度 | 说明 |
|---|---|
| **业务定位** | 代理佣金管理系统（CPA, Cost Per Action） |
| **核心目标** | 替换第三方 Cellxpert，把数据控制权拿回公司 |
| **业务领域** | Forex / CFD 外汇与差价合约 |
| **使用方** | 公司运营（ToB） · 代理 IB（ToC） · 内部 Job（ToJ） · 对外开放 API（ToO） |
| **架构特征** | 4 个独立业务域 + 4 层严格分层 |
| **多品牌支持** | 启动参数 `-Dbrand=AU` 等区分品牌 |
| **技术栈** | Spring Boot 3 · Spring Cloud · Consul · OpenFeign · MyBatis · Redis/Redisson · QlExpress · MapStruct · Xxl-Job · Kafka · Knife4j |

---

## 4 大业务领域

| 领域 | 模块 | 端口 | 用途 |
|---|---|---|---|
| **ToB** Admin | `hytech-cpa-admin` | 9101 | 运营/管理员后台 |
| **ToC** Client | `hytech-cpa-client` | 9103 | 代理 IB 自助门户 |
| **ToJ** Job | `hytech-cpa-job` | 9105 | 定时任务 / MQ 消费（Xxl-Job + Kafka） |
| **ToO** Open | `hytech-cpa-open` | 9106 | 对外开放 API |

> ⚠️ **严禁跨域服务调用**。领域间通过物理/逻辑网络隔离，仅能通过标准化 `*-platform-api` 数据契约交互。

---

## 9 大业务模块速查

| # | 业务模块 | 入口（典型 Controller / Service） |
|---|---|---|
| 1 | 代理账户管理 | `BsnCpaAccountController`、`BsnCpaUpgradeController` |
| 2 | 客户管理 | `BsnCpaClientController`、`BsnClientRelationController`、`BsnTransferClientController` |
| 3 | 佣金规则管理 | `BsnRuleGroupController`、`BsnTierController`、`BsnSymbolGroupController`、`BsnRuleCollocateController` |
| 4 | 协议签署 | `BsnRuleAgreementController`（admin + client 两端） |
| 5 | **佣金计算（核心）** | `BonusCalculationCoreService` + `EventHandleCoreService` + 4 个 `BonusCalculator` 策略实现 |
| 6 | 多层级分润 | `BonusShareCoreService` + `RuleBonusShareEntity` |
| 7 | 结算与支付 | `BsnSettlementController`、`BsnBonusApproveController` |
| 8 | BPM 审批 | `hytech-cpa-bpm` 模块（含 `ApproveProcess` / `ApproveWork` / `ApproveTask`） |
| 9 | 报告统计 | `BsnCpaReportController`、`BsnCpaBonusShareDayController` |

---

## 工程模块全景

```
hytech-cpa/
├── hytech-cpa-common          # 通用工具：枚举、常量、JSON 等
├── hytech-cpa-core-backend    # 核心数据访问层：Entity / Mapper / CoreService
├── hytech-cpa-backend         # 公共 backend API（小型外提服务）
├── hytech-cpa-bpm             # 审批流程引擎
├── hytech-cpa-admin           # ToB 后台（运营/管理员）
├── hytech-cpa-client          # ToC 代理自助门户
│   └── cpa-client-read-service    # 读写分离的只读服务
├── hytech-cpa-open            # ToO 对外开放 API
├── hytech-cpa-center          # 中心调度
├── hytech-cpa-job             # ToJ 定时任务执行器
├── hytech-cpa-alert           # Lark（飞书）告警通知模块
├── hytech-cpa-leaf            # 美团 Leaf 分布式 ID（高性能 ID 生成）
├── hytech-cpa-model           # 模型对象（公共 model）
└── hytech-cpa-data-sync       # 数据同步（Cellxpert 迁移期使用）
```

---

## 知识文档索引

> 详细内容按需阅读 `docs/` 下的子文档，**不要一次性全部加载，按需触发**。

| 文档 | 何时阅读 |
|---|---|
| [01-architecture.md](docs/01-architecture.md) | 理解整体架构、4 领域、4 层分层 |
| [02-business-modules.md](docs/02-business-modules.md) | 修改某业务模块前必读 |
| [03-data-model.md](docs/03-data-model.md) | 写 Mapper / SQL / 表设计相关代码前 |
| [04-tech-stack.md](docs/04-tech-stack.md) | 技术栈细节、Redis/Kafka 规范、告警模块 |
| [05-development-guide.md](docs/05-development-guide.md) | 写任何代码前都建议读 — 包含分层硬规则与命名 |
| [06-business-flows.md](docs/06-business-flows.md) | 排查佣金/分润/结算异常时阅读 |
| [07-faq.md](docs/07-faq.md) | 常见问题、故障排查路径 |
| [08-onboarding.md](docs/08-onboarding.md) | 新员工入职第一周阅读 |

---

## 关键硬规则（必须遵守）

1. **不跨领域直接调用**：ToB 服务不能直接调用 ToC 服务的 Controller/Service
2. **backend 不返回 PO**：backend 层禁止直接对外返回 DB 实体（Entity），必须用 DTO
3. **business 不直接访问其他业务数据库**：仅能通过本业务 backend 操作本业务库
4. **接口前缀强约束**：backend 层接口必须以 `/bck/` 开头，business 层必须以 `/bsn/` 开头
5. **Redis Key 规范**：`域:业务:模块:功能:数据`，全小写，如 `partner:cpa:leads:info:{leadsId}`
6. **Kafka Topic 规范**：`域.业务.模块.功能`，全小写，如 `partner.cpa.leads.allocate`

详细规则与示例见 [05-development-guide.md](docs/05-development-guide.md)。

---

## 关联 Skill

安装本 Skill 时，会自动级联安装：

- `dev-standards` — 公司通用研发规范
- `karpathy-guidelines` — AI 编码行为约束（避免 AI 把代码写崩）

---

## 元信息

| 字段 | 值 |
|---|---|
| Owner | hytech-cpa 研发团队 |
| 主要负责人 | Tech Lead |
| 维护频率 | 每个迭代结束更新 |
| 适用品牌 | AU 等多品牌（通过 `-Dbrand` 区分） |
| 替换的旧系统 | Cellxpert（第三方 SaaS） |
