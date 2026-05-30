# 02 · 9 大业务模块详解

> 本文档以「业务入口 → 核心实体 → 关键流程 → 注意事项」结构，逐个讲清 9 大业务模块。
> 改任何业务代码前先定位到对应模块。

---

## 模块 1 · 代理账户管理

### 业务定义
管理代理账户（IB/CPA/Partner）的注册、状态、上下级关系、升级、转移。

### 入口 Controller

| Controller | 位置 | 用途 |
|---|---|---|
| `BsnCpaAccountController` | admin/business | 代理账户概要、个人信息、笔记、可用余额 |
| `BsnCpaUpgradeController` | admin/business | 代理升级申请管理 |
| `BsnCpaSubController` | admin/business | 子代理管理 |
| `BsnRelationMaintainController` | admin/business | 上下级关系维护 |
| `BsnTransferCpaController` | admin/business | 代理转移（如换团队） |
| `BsnAutoAllocManagerController` | admin/business | 自动分配规则配置 |
| `BsnCpaUserInfoController` | client/business | 代理自助查看个人信息 |
| `BsnCpaRegisterController` | client/business | 代理自助注册 |
| `BsnAccountInvitationCodeController` | client/business | 邀请码管理 |

### 核心实体

```
CpaAccountEntity (tb_p_cpa_account)
├── userId           ← 对应 DAP/User 服务的 userId
├── account          ← 系统内唯一代理账号（Long 数值 ID）
├── accountType      ← AGENT / SUB_AGENT / PARTNER（枚举 CpaAccountTypeEnum）
├── accountStatus    ← VALID / INVALID / SUSPENDED（枚举 CpaAccountStatusEnum）
├── accountCategory  ← GENERAL（普通）/ SYSTEM（系统）
├── registerTime
└── selected         ← 0-否 1-是

CpaUserEntity (tb_p_cpa_user)        ← 代理用户基本信息
CpaAccountRelationEntity (tb_p_cpa_account_relation)  ← 上下级关系
├── masterAccount    ← 顶级主账户
├── cpaAccount       ← 当前账户
├── parentCpaAccount ← 直属上级账户
└── bindTime

CpaAccountPathEntity (tb_p_cpa_account_path)  ← 完整路径（冗余加速查询）
└── path             ← /master_id/level2_id/level3_id/
```

### 注意事项

- **path 字段**是为了快速查询某账户的所有上下级，**变更关系时务必同步更新**
- 同账户禁止在系统中重复注册，account 字段是唯一索引
- 升级流程（普通代理 → 子代理 → 主代理）走 `UpgradeCpaApplyEntity`，需 BPM 审批

---

## 模块 2 · 客户管理

### 业务定义
管理代理名下的客户（即终端交易用户）、绑定关系、状态变化。

### 入口 Controller

| Controller | 用途 |
|---|---|
| `BsnCpaClientController` | 客户列表查询 |
| `BsnClientRelationController` | 客户关系维护 |
| `BsnTransferClientController` | 客户转移（在代理间迁移） |
| `BsnClientManageForSWSController` | SWS 专用客户管理 |
| `BsnCpaCrmController` | CRM 数据集成 |
| `BsnClientManagementController` | client 端客户管理 |
| `BsnMarketChannelController` | 营销渠道管理 |

### 核心实体

```
ClientEntity (tb_p_client)
├── userId             ← 对应 DAP/User 的 userId
├── clientStatus       ← REGISTER（已注册无账号）/ LIVE（已开账号）/ QUALIFY（满足 QFTD）
├── inviterCpaAccount  ← 首次邀请此客户的代理账号
├── registerTime
├── liveTime           ← 开账号时间
├── qualifyTime        ← 达到 QFTD 条件的时间
├── regular            ← 所属监管机构
└── license            ← 监管执照

AccountClientRelationEntity (tb_p_account_client_relation)
├── clientUserUid       ← 客户的 userId
├── parentCpaAccount    ← 归属代理账户 ID
├── parentCpaUserId     ← 归属代理用户 ID
├── bindTime
├── valid               ← 1=生效 / 0=失效
└── clientStatus        ← 同 ClientEntity

CrmUserEntity (tb_p_crm_user)       ← 与外部 CRM 同步的用户信息
```

### 状态机

```
REGISTER ──首次入金──> LIVE ──满足 QFTD──> QUALIFY
                                   │
                                   └──> 触发佣金计算
```

### 注意事项

- `valid` 字段标记关系是否有效（客户转移时旧关系 valid=0，新关系 valid=1）
- 同一客户在同一时刻只能有 1 条 valid=1 的关系
- `BsnTransferClientController` 转移操作会同步更新 valid 标识 + 创建新关系

---

## 模块 3 · 佣金规则管理

### 业务定义
配置佣金计算规则：什么客户合格（QFTD）、按什么模式（FTD/Country/Progressive）计算、分多少级（Tier）、按什么周期结算。

### 入口 Controller

| Controller | 用途 |
|---|---|
| `BsnRuleGroupController` | 规则组管理 |
| `BsnTierController` | Tier 分级配置 |
| `BsnTierCountryController` | 按国家细分 Tier |
| `BsnSymbolGroupController` | 交易品种分组（影响渐进式规则） |
| `BsnRuleExtraBonusController` | 额外奖励规则 |
| `BsnRuleCollocateController` | 规则关联代理 |
| `BsnRuleController`（client） | 代理自助查看规则 |

### 核心实体链

```
RuleGroupEntity（规则组 tb_p_rule_group）
  └── RuleEntity（规则主体 tb_p_rule）
        ├── id
        ├── ruleType            ← group / cpa / default
        ├── settlementCycle     ← week / biweek / month
        ├── commissionType      ← by_ftd / by_country / deactivate_cpa / progressive_cpa_tiers
        ├── multipleEnable      ← 多次 QFTD 开关
        ├── customerRange       ← 0=增量 / 1=全量
        ├── agreementModify     ← 0=不需要更新协议 / 1=需要更新
        └── rangeDateStart/End  ← 应用范围时间窗
        │
        ├── RuleQftdConditionEntity     ← QFTD 条件（"合格入金"标准）
        ├── TierEntity                  ← 不同金额段对应不同佣金
        │   └── TierCountryEntity       ← 按国家细分
        ├── SymbolGroupEntity           ← 交易品种分组
        ├── RuleExtraBonusEntity        ← 额外奖励规则
        │   └── RuleExtraBonusItemEntity ← 奖励明细
        └── RuleCpaEntity               ← 规则与具体代理的映射
```

### 佣金模式（CommissionType）

| 模式 | 说明 | 计算器实现类 |
|---|---|---|
| `by_ftd` (FTD) | 首次入金阶梯奖励 | `FTDBonusCalculator` |
| `by_country` (Country) | 按客户国家计算 | `CountryBonusCalculator` |
| `progressive_cpa_tiers` (Progressive) | 渐进式阶梯（多次入金累加） | `ProgressiveBonusCalculator` |
| `deactivate_cpa` (Deactivate) | 停用状态规则 | — |

> 计算器采用**策略模式**，所有 `BonusCalculator` 实现类注册到 `bonusCalculatorMap`，按 `CommissionTypeEnum` 路由。

### 结算周期（SettlementCycle）

- `WEEK` 每周
- `BIWEEK` 双周
- `MONTH` 每月

### 注意事项

- 规则变更（如阶梯金额调整）通常需要重新签署协议（`agreementModify=1`）
- 已签署的协议会保留**规则快照**（`ruleSnapshot` 字段），即使规则后续修改也以快照为准
- 新增佣金模式时：① 新增 `CommissionTypeEnum` 枚举 ② 实现 `BonusCalculator` 接口 ③ Spring 会自动注册

---

## 模块 4 · 协议签署

### 业务定义
代理必须签署协议后规则才能生效，协议中保存规则快照。

### 入口 Controller

| Controller | 位置 | 用途 |
|---|---|---|
| `BsnRuleAgreementController` | admin/business | 后台管理协议 |
| `BsnRuleAgreementController` | client/business | 代理自助签署 |

### 核心实体

```
AgreementEntity (tb_p_agreement)
├── cpaUserId
├── cpaAccount
├── ruleId           ← 绑定的规则 ID
├── groupId          ← 规则组 ID
├── agreementStatus  ← I（未签）/ S（已签）
├── signedTime
└── ruleSnapshot     ← 签署时刻的规则内容 JSON ⚡（关键）

DefaultAgreementFileEntity   ← 默认协议文件（PDF 等）
AgreementLogEntity           ← 协议状态变更日志
```

### 注意事项

- **规则快照机制**：即使后续规则修改，已签协议仍按签署时的快照计算佣金
- 当 `agreementModify=1` 时，规则修改会让该规则下所有代理协议状态变为"待重签"
- 签署时机：通常是代理首次进入 client 端 → 弹出协议确认页 → 签署后激活

---

## 模块 5 · 佣金计算（最核心）

### 业务定义
根据客户事件（入金 / 交易）触发佣金计算，落入佣金记录表。

### 触发链

```
客户出入金事件（来自 MT4/MT5）
  → EventPaymentEntity (tb_p_event_payment)
  → 检查 QFTD 条件（ConditionCoreService.qftdQualified）
  → BonusQftdEntity (tb_p_bonus_qftd) 记录 QFTD 触发
  → 匹配规则 → 调度对应 BonusCalculator → 计算佣金
  → BonusExtraEntity (tb_p_bonus_extra) 额外奖励记录
  → 生成结算单 SettlementEntity

客户交易事件
  → EventTradeEntity (tb_p_event_trade)
  → 参与渐进式 / 品种分组规则计算
```

### 核心 Service

| Service | 位置 | 用途 |
|---|---|---|
| `EventHandleCoreService` | core-backend | 事件入口，串联整个流程 |
| `BonusCalculationCoreService` | core-backend | 佣金计算调度（路由到具体 Calculator） |
| `ConditionCoreService` | core-backend | 条件判断（QFTD、ExtraBonus） |
| `BonusCalculator`（接口） | core-backend | 4 个实现：FTD / Country / Progressive / Deactivate |
| `BonusShareCoreService` | core-backend | 多层级分润计算 |
| `BonusTransactional` | core-backend | 整个流程的事务包装 |

### 核心实体

```
EventPaymentEntity (tb_p_event_payment)  ← 入金事件
EventTradeEntity (tb_p_event_trade)      ← 交易事件
BonusQftdEntity (tb_p_bonus_qftd)        ← QFTD 触发记录
BonusExtraEntity (tb_p_bonus_extra)      ← 额外奖励
BonusExtraItemEntity                     ← 奖励明细
ClientSummaryEntity (tb_p_client_summary) ← 客户摘要汇总
```

### 关键代码片段（FTDBonusCalculator）

```java
@Component
public class FTDBonusCalculator implements BonusCalculator {

    @Override
    public CommissionTypeEnum getType() { return CommissionTypeEnum.FTD; }

    @Override
    public void calculate(ClientSummaryDto clientSummary) {
        List<RuleQftdBonusItemDto> ruleQftdBonusItems = clientSummary.getRuleQftdBonusItems();

        // 找命中的 Tier（ftdAmount 落在 [fromAmount, toAmount) 区间）
        List<RuleQftdBonusItemDto> hitItems = ruleQftdBonusItems.stream()
                .filter(item -> {
                    BigDecimal ftdAmount = clientSummary.getFtdAmount();
                    return ftdAmount.compareTo(item.getFromAmount()) >= 0
                        && ftdAmount.compareTo(item.getToAmount()) < 0;
                }).toList();

        if (hitItems.size() > 1) {
            // ⚡ 异常：命中多条规则 → 发飞书告警
            larkCommMessagePusher.sendAlert(LarkAlertRequest.builder()
                .title("佣金计算").description("ftd type qualified multiple bonus items.")
                .build());
            return;
        }

        BonusQftdDto bonusQftdDto = new BonusQftdDto(clientSummary, hitItems.get(0), null, ...);
        clientSummary.doCompletedBonus(bonusQftdDto, true);
    }
}
```

### 注意事项

- **并发控制**：佣金计算入口加 RedissonLock，按 `clientUserId` 维度
- **告警驱动排查**：FTD 命中多条规则、规则不存在等异常都会发飞书告警，标题：`佣金计算 / 规则错误`
- **MDC 链路追踪**：`MDC.put("eventid", UUID.randomUUID().toString())`，整个事件处理链路打通日志
- **Kafka 事件**：QFTD / Bonus 等关键事件会发 Kafka，topic 见 `TopicConstant`

---

## 模块 6 · 多层级分润

### 业务定义
当下级代理产生佣金时，上级（多层）按配置比例获得分润。

### 核心实体

```
RuleBonusShareEntity（分润规则 tb_p_rule_bonus_share）
├── cpaAccount       ← 配置分润的代理账户（顶级）
└── shareLevel       ← 分润层级数（如 3 层）

RuleBonusShareItemEntity（分润规则项）
├── childLevel       ← 第几级下级（1 = 直属，2 = 隔一级）
├── childAccount     ← 下级代理账户
└── shareRate        ← 分配比例（0 ~ 1，如 0.10 表示 10%）

BonusShareEntity（实际分润记录 tb_p_bonus_share）
├── masterAccount    ← 主账户（产生佣金的代理）
├── parentAccount    ← 上级（分润受益方）
├── childAccount     ← 子级（原始佣金产生方）
├── bonusAmount      ← 原始佣金
└── shareAmount      ← 分润金额

BonusShareDayEntity（日度汇总 tb_p_bonus_share_day）
├── bonusDate
├── cpaAccount
├── clientCount      ← 当日客户数
├── depositAmount    ← 当日入金
├── bonusAmount      ← 当日佣金
└── shareAmount      ← 当日分润
```

### 计算示例

```
顶级代理 A 配置分润规则：
  - 一级下级分 10%（rate=0.10）
  - 二级下级分 5%（rate=0.05）

当 B（A 的二级下级）的客户产生 $100 佣金：
  B 收到 $100（原始佣金）
  B 的直属上级 C 收到 $5（$100 × 5%）
  C 的直属上级 A 收到 $10（$100 × 10%）
```

### 入口

- 后台配置：`BsnRuleGroupController` 的分润规则子页面
- 代理查看：`BsnBonusShareController`（client）
- 核心服务：`BonusShareCoreService.calculateShare(BonusEntity)`

### 注意事项

- 分润计算在主佣金计算完成后**异步触发**（runAsync）
- 分润链路深度查询走 `CpaAccountPathEntity.path` 字段加速
- 日度汇总由 `cpa-job-executor` 中的 `BonusShareDayJob` 凌晨跑批

---

## 模块 7 · 结算与支付

### 业务定义
定期把佣金汇总成结算单，代理申请出金，BPM 审批后实际支付。

### 流程

```
1. 定期（周/双周/月）自动汇总 → SettlementEntity 结算单
2. 代理查看可申请金额    → AccountBalanceEntity 余额
3. 代理提交出金申请      → ApplyPaymentEntity（状态 P：审批中）
4. BPM 审批              → ApproveWorkEntity
5. 审批通过 → 执行支付
6. 更新余额              → AccountBalanceRecordEntity（变动记录）
```

### 入口 Controller

| Controller | 位置 | 用途 |
|---|---|---|
| `BsnSettlementController` | admin/business | 后台结算单管理 |
| `BsnSettlementController` | client/business | 代理查询结算 |
| `BsnBonusApproveController` | admin/business | 出金审批 |

### 核心实体

```
SettlementEntity (tb_p_c_settlement)
├── settlementNo     ← 结算单号
├── cpaAccount
├── clientUserId     ← 对应客户
├── settlementAmount
├── currency
├── settlementType   ← FTD / Country / Progressive / Bonus / ExtraBonus / Deduction
├── autoFlag         ← 0=手动 / 1=自动
├── startDate
├── endDate
└── bonusTime        ← 佣金产生时间

ApplyPaymentEntity (tb_p_c_apply_payment)
├── payNo            ← 付款单号
├── cpaAccount
├── payAmount        ← 申请金额
├── realPayAmount    ← 实际支付金额
├── applyStatus      ← P（审批中）/ S（同意）/ F（拒绝）
└── payTime

AccountBalanceEntity (tb_p_c_account_balance)
├── account
├── availableBalance ← 可用余额
├── lockedBalance    ← 锁定余额（待审批中的金额）
└── currency

AccountBalanceRecordEntity   ← 余额变动流水
ApplyPaymentDetailEntity     ← 出金申请明细
```

### 状态机

```
ApplyPaymentEntity.applyStatus:
   ┌──────────┐
   │    P     │ ── 审批通过 ──> S
   │ 审批中    │ ── 审批拒绝 ──> F
   └──────────┘
```

### 注意事项

- **锁定余额机制**：提交出金时 `availableBalance` 减少，`lockedBalance` 增加；通过后两者一起释放
- **结算单号迁移规则**：从 Cellxpert 迁移的历史数据加前缀（如 `CX-`）区分
- 自动结算任务：`cpa-job-executor/payment/AccountBalanceJob`

---

## 模块 8 · BPM 审批流程

### 业务定义
所有重要操作（出金、规则修改、代理升级等）都走统一的 BPM 流程。

### 核心实体

```
ApproveProcessEntity   ← 流程定义模板（如"出金审批流程"）
ApproveNodeEntity      ← 流程节点（每个节点的审批人配置）
ApproveWorkEntity      ← 流程实例（一次具体审批）
ApproveTaskEntity      ← 审批任务（每个节点的待办任务）
ApproveRecordEntity    ← 审批历史记录（谁在何时做了什么）
```

### 流程状态

```
0=审批中 → 1=通过 / 2=拒绝 / 3=撤销 / 4=超时终止
```

### 入口 Controller

| Controller | 用途 |
|---|---|
| `BsnApproveController` | 通用审批操作（同意/拒绝/转交/撤回） |
| `BsnBonusApproveController` | 佣金/出金相关的审批 |

### 注意事项

- BPM 模块位于独立的 `hytech-cpa-bpm` Module
- 业务侧只需调用 `bpmService.startWork(processCode, businessId, ...)` 启动流程
- 审批人配置在 `ApproveNodeEntity`，支持按角色、按账号配置
- 超时自动终止：定时任务扫描 `ApproveWorkEntity` 中超时的工作项

---

## 模块 9 · 报告统计

### 业务定义
为运营、代理、管理员提供各种维度的报告。

### 报告维度

| 报告 | 描述 |
|---|---|
| 业绩报告（Performance Report） | 代理名下客户的出入金、交易量汇总 |
| 出入金报告（Deposit/Withdrawal） | 明细出入金记录 |
| 佣金报告 | 佣金计算明细 |
| 分佣报告 | 分润明细 |

### 入口 Controller

| Controller | 用途 |
|---|---|
| `BsnCpaReportController` | 代理业绩报告主入口（admin） |
| `BsnCpaBonusShareDayController` | 日度分润查询 |

### 核心服务

| Service | 位置 | 用途 |
|---|---|---|
| `DailyReportService` | job/executor | 每日统计汇总（凌晨跑批） |
| `PerformanceReportServiceImpl` | admin/business | 业绩报告查询 |

### 核心统计表

```
BonusShareDayEntity (tb_p_bonus_share_day)         ← 日度分润统计
AdminBonusShareDayEntity (tb_p_admin_bonus_share_day) ← 管理员视角日度统计
ClientSummaryEntity (tb_p_client_summary)          ← 客户摘要汇总（总入金、总交易量）
```

### 注意事项

- 大查询走 `cpa-client-read-service` 只读服务，避免影响主库
- 业绩报告字段较多，新增字段时同步看 `docs/overview-performance-new-fields.md`
- 导出使用 EasyExcel，定义 `@ExcelProperty` 注解
