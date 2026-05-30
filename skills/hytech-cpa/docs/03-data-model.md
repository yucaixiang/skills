# 03 · 核心数据模型

> 本文档汇总 hytech-cpa 核心数据库表、字段、关系，配合开发 Mapper / SQL / Entity 时使用。
> 表前缀：`tb_p_` = 业务核心表 · `tb_p_c_` = 结算/支付表 · `tb_p_event_` = 事件表 · `tb_p_bonus_` = 佣金表。

---

## 一、Entity 通用规约

所有 Entity 继承 `BaseEntity`：

```java
@Getter @Setter
@SuperBuilder @AllArgsConstructor @NoArgsConstructor
@ToString(callSuper = true)
@EqualsAndHashCode(callSuper = true)
public class XxxEntity extends BaseEntity {
    // 业务字段
}
```

`BaseEntity` 通用字段：
```
id              主键
createBy        创建人
createTime      创建时间
updateBy        更新人
updateTime      更新时间
brand           品牌（多品牌隔离）
deleted         软删标记
```

实体类位置：`com.hytech.cpa.core.backend.model.db.{业务包}`

---

## 二、核心实体全景图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          代理账户域 (user 包)                            │
│                                                                          │
│   CpaAccountEntity ──┬─→ CpaUserEntity                                   │
│   (代理账户主表)      ├─→ CpaAccountRelationEntity (上下级关系)           │
│                      ├─→ CpaAccountPathEntity (完整路径，加速查询)       │
│                      └─→ UpgradeCpaApplyEntity (升级申请)                │
│                                                                          │
│   ClientEntity ──────┬─→ AccountClientRelationEntity (客户绑代理)        │
│   (客户主表)          └─→ CrmUserEntity (与 CRM 同步)                    │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          佣金规则域 (bonus/rule 包)                      │
│                                                                          │
│   RuleGroupEntity ──→ RuleEntity ──┬─→ RuleQftdConditionEntity           │
│   (规则组)            (规则主体)    ├─→ RuleQftdBonusItemEntity (合格档位)│
│                                    ├─→ TierEntity (阶梯) ──→ TierCountry │
│                                    ├─→ SymbolGroupEntity (品种分组)       │
│                                    ├─→ RuleExtraBonusEntity              │
│                                    │     └─→ RuleExtraBonusItemEntity    │
│                                    ├─→ RuleCpaEntity (规则关联代理)       │
│                                    └─→ AgreementEntity (协议+快照)        │
│                                                                          │
│   RuleBonusShareEntity (分润规则) ─→ RuleBonusShareItemEntity            │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ 触发计算
┌──────────────────────────────────────────────────────────────────────────┐
│                          佣金计算域 (bonus/calculate, bonus/share 包)    │
│                                                                          │
│   EventPaymentEntity (入金事件) ──┐                                       │
│   EventTradeEntity (交易事件)   ──┼──→ ClientSummaryEntity (客户汇总)    │
│                                    │     │                               │
│                                    │     ├─→ BonusQftdEntity (QFTD 记录) │
│                                    │     └─→ BonusExtraEntity (额外奖励) │
│                                    │           └─→ BonusExtraItemEntity  │
│                                    │                                     │
│                                    └──→ BonusShareEntity (分润记录)      │
│                                          └─→ BonusShareDayEntity (日汇总)│
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ 结算
┌──────────────────────────────────────────────────────────────────────────┐
│                          结算支付域 (payment 包)                          │
│                                                                          │
│   SettlementEntity (结算单) ──→ ApplyPaymentEntity (出金申请)             │
│                                  └─→ ApplyPaymentDetailEntity (明细)     │
│                                                                          │
│   AccountBalanceEntity (账户余额) ──→ AccountBalanceRecordEntity (流水)  │
└──────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ 审批
┌──────────────────────────────────────────────────────────────────────────┐
│                    BPM 审批域 (hytech-cpa-bpm 模块)                       │
│                                                                          │
│   ApproveProcessEntity ──→ ApproveNodeEntity (流程节点)                  │
│   (流程定义模板)                                                          │
│        │                                                                  │
│        ▼ 触发                                                             │
│   ApproveWorkEntity (流程实例) ──→ ApproveTaskEntity (审批任务)          │
│                                     └─→ ApproveRecordEntity (审批记录)   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 三、代理账户域

### 3.1 `tb_p_cpa_account` · 代理账户主表

```java
public class CpaAccountEntity extends BaseEntity {
    private Long userId;            // 对应 DAP/User 服务的 userId
    private String accountType;     // CPA / IB / AFFILIATE（CpaAccountTypeEnum）
    private Long account;           // 系统内唯一代理账号（Long 数值 ID）
    private String accountStatus;   // VALID / INVALID / SUSPENDED（CpaAccountStatusEnum）
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime registerTime;
    private String remark;
    private Integer selected;       // 0=否 1=是
    private String accountCategory; // GENERAL（普通）/ SYSTEM（系统）
}
```

**索引建议**：
- `idx_user_id` · 按 userId 查询
- `idx_account` · 按账号查询（唯一）
- `idx_status` · 按状态过滤

### 3.2 `tb_p_cpa_account_relation` · 上下级关系

```java
public class CpaAccountRelationEntity extends BaseEntity {
    private Long masterAccount;      // 顶级主账户
    private Long cpaAccount;         // 当前账户
    private Long parentCpaAccount;   // 直属上级账户
    private LocalDateTime bindTime;
}
```

### 3.3 `tb_p_cpa_account_path` · 账户路径（冗余表）

```java
public class CpaAccountPathEntity extends BaseEntity {
    private Long cpaAccount;
    private String path;  // 格式：/master_id/level2_id/level3_id/
}
```

**作用**：用 `LIKE '/master_id/%'` 快速查询某主账户下所有下级（不需要递归查询关系表）。

### 3.4 `tb_p_cpa_user` · 代理用户基本信息

包含代理的个人信息（姓名、邮箱、手机、KYC 状态等），通过 `userId` 与 `CpaAccountEntity` 关联。

### 3.5 `tb_p_upgrade_cpa_apply` · 升级申请

代理从 AFFILIATE → IB → CPA 的升级申请记录，状态走 BPM 审批。

---

## 四、客户域

### 4.1 `tb_p_client` · 客户主表

```java
public class ClientEntity extends BaseEntity {
    private Long userId;
    private String clientStatus;       // REGISTER / LIVE / QUALIFY（ClientStatusEnum）
    private Long inviterCpaAccount;    // 首次邀请此客户的代理
    private LocalDateTime registerTime;
    private LocalDateTime liveTime;    // 开账号时间
    private LocalDateTime qualifyTime; // 达到 QFTD 条件的时间
    private String regular;            // 所属监管机构
    private String license;            // 监管执照
}
```

**状态机**：
```
REGISTER ──首次入金──> LIVE ──满足 QFTD 条件──> QUALIFY
                                                  │
                                                  └──> 触发佣金计算
```

### 4.2 `tb_p_account_client_relation` · 客户绑定代理

```java
public class AccountClientRelationEntity extends BaseEntity {
    private Long clientUserUid;        // 客户的 userId
    private Long parentCpaAccount;     // 归属代理账户
    private Long parentCpaUserId;      // 归属代理用户
    private LocalDateTime bindTime;
    private Integer valid;             // 1=生效 / 0=失效
    private String clientStatus;       // 同 ClientEntity
}
```

**重要约束**：同一客户在同一时刻**只能有 1 条 `valid=1` 的关系**。客户转移：旧关系 `valid=0`，新建一条 `valid=1`。

### 4.3 `tb_p_crm_user` · CRM 同步用户

通过 Feign 同步外部 CRM 用户数据。`UserSyncCoreServiceImpl` 比对差异并更新。

---

## 五、佣金规则域

### 5.1 `tb_p_rule_group` · 规则组

```java
public class RuleGroupEntity extends BaseEntity {
    private String groupName;
    private String groupStatus;
    // ...
}
```

### 5.2 `tb_p_rule` · 规则主体

```java
public class RuleEntity extends BaseEntity {
    private Long id;
    private String ruleType;          // group / cpa / default
    private String settlementCycle;   // week / biweek / month
    private String commissionType;    // by_ftd / by_country / deactivate_cpa / progressive_cpa_tiers
    private Boolean multipleEnable;   // 多次 QFTD 开关
    private Boolean customerRange;    // 0=增量 / 1=全量
    private Boolean agreementModify;  // 0=不需更新协议 / 1=需要更新
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate rangeDateStart;
    @JsonFormat(pattern = "yyyy-MM-dd")
    private LocalDate rangeDateEnd;
}
```

### 5.3 `tb_p_rule_qftd_condition` · QFTD 合格条件

定义"什么样的入金算合格"，比如：
- 入金 ≥ $100
- 且开账号后 90 天内入金
- 且交易量 ≥ 1 手

### 5.4 `tb_p_rule_qftd_bonus_item` · QFTD 合格档位

```
fromAmount | toAmount  | bonusAmount
-----------+-----------+--------------
$100       | $500      | $20
$500       | $1000     | $50
$1000      | $5000     | $150
$5000      | ∞         | $500
```

代码中按 `ftdAmount ∈ [fromAmount, toAmount)` 查找命中。

### 5.5 `tb_p_tier` · 阶梯配置（Country / Progressive 模式用）

按代理累计指标（客户数、入金量等）分级。

### 5.6 `tb_p_tier_country` · 按国家细分

不同国家给不同档位（CountryBonusCalculator 用）。

### 5.7 `tb_p_symbol_group` · 交易品种分组

将 MT4/MT5 上千个交易品种分组（如 "Forex 主流"、"Crypto"、"商品"），不同分组佣金不同（Progressive 模式用）。

### 5.8 `tb_p_rule_extra_bonus` · 额外奖励规则

额外的"达成 ROI / 客户数"奖励。

```java
RuleExtraBonusEntity:
    private BigDecimal roi;          // ROI 阈值

RuleExtraBonusItemEntity:
    private Integer qualifiedMin;    // 合格客户数下限
    private Integer qualifiedMax;    // 合格客户数上限
    private BigDecimal bonusAmount;  // 奖励金额
```

### 5.9 `tb_p_rule_cpa` · 规则关联代理

规则不会"全员适用"，需明确关联到具体代理：
```java
RuleCpaEntity:
    private Long ruleId;
    private Long cpaAccount;
    private LocalDate effectiveDate;
```

### 5.10 `tb_p_agreement` · 协议表

```java
public class AgreementEntity extends BaseEntity {
    private Long cpaUserId;
    private Long cpaAccount;
    private Long ruleId;
    private Long groupId;
    private String agreementStatus;  // I（未签）/ S（已签）
    private LocalDateTime signedTime;
    private String ruleSnapshot;     // ⚡ 关键：签署时刻的规则 JSON 快照
}
```

**关键设计**：`ruleSnapshot` 字段保存协议签署时的完整规则 JSON。**即使规则后续修改，已签协议仍按快照计算佣金**。

---

## 六、分润域

### 6.1 `tb_p_rule_bonus_share` · 分润规则

```java
RuleBonusShareEntity:
    private Long cpaAccount;    // 配置分润的代理（通常是顶级主代理）
    private Integer shareLevel; // 分润层级数
```

### 6.2 `tb_p_rule_bonus_share_item` · 分润规则项

```java
RuleBonusShareItemEntity:
    private Long ruleBonusShareId;
    private Integer childLevel;    // 第几级下级（1 = 直属下级）
    private Long childAccount;
    private BigDecimal shareRate;  // 0 ~ 1（0.10 = 10%）
```

### 6.3 `tb_p_bonus_share` · 实际分润记录

```java
BonusShareEntity:
    private Long masterAccount;    // 主账户（顶级）
    private Long parentAccount;    // 上级（分润受益方）
    private Long childAccount;     // 子级（原始佣金产生方）
    private BigDecimal bonusAmount;  // 原始佣金
    private BigDecimal shareAmount;  // 分润金额
    private LocalDate bonusDate;
```

### 6.4 `tb_p_bonus_share_day` · 日度汇总

```java
BonusShareDayEntity:
    private LocalDate bonusDate;
    private Long cpaAccount;
    private Integer clientCount;
    private BigDecimal depositAmount;
    private BigDecimal bonusAmount;
    private BigDecimal shareAmount;
```

---

## 七、佣金计算域

### 7.1 `tb_p_event_payment` · 入金事件

```java
EventPaymentEntity:
    private Long clientUserId;
    private BigDecimal amount;
    private String currency;
    private LocalDateTime eventTime;
    private String paymentType;   // DEPOSIT / WITHDRAW
    private String dataSource;    // MT4 / MT5
```

### 7.2 `tb_p_event_trade` · 交易事件

```java
EventTradeEntity:
    private Long clientUserId;
    private String symbol;        // 交易品种
    private BigDecimal lots;      // 交易手数
    private BigDecimal commission;
    private LocalDateTime eventTime;
    private String dataSource;
```

### 7.3 `tb_p_bonus_qftd` · QFTD 记录

```java
BonusQftdEntity:
    private Long clientUserId;
    private Long cpaAccount;
    private Long ruleId;
    private Long bonusItemId;     // 命中的档位 ID
    private BigDecimal bonusAmount;
    private LocalDateTime bonusTime;
    private String ruleSnapshot;  // 计算时使用的规则快照
```

### 7.4 `tb_p_bonus_extra` · 额外奖励记录

```java
BonusExtraEntity:
    private Long cpaAccount;
    private BigDecimal roi;
    private Integer qualifiedCount;
    private BigDecimal bonusAmount;
    private LocalDate bonusDate;
```

### 7.5 `tb_p_client_summary` · 客户摘要汇总

聚合表，保存每个客户的：
- 总入金 / 总出金
- 总交易量
- QFTD 是否达成、何时达成
- 是否已生成佣金、佣金金额

**这是计算的"工作台"实体**，所有 BonusCalculator 都基于 `ClientSummaryDto` 操作。

---

## 八、结算支付域

### 8.1 `tb_p_c_settlement` · 结算单

```java
SettlementEntity:
    private String settlementNo;       // 结算单号
    private Long cpaAccount;
    private Long clientUserId;
    private BigDecimal settlementAmount;
    private String currency;
    private String settlementType;     // FTD / Country / Progressive / Bonus / ExtraBonus / Deduction
    private Integer autoFlag;          // 0=手动 / 1=自动
    private LocalDate startDate;
    private LocalDate endDate;
    private LocalDateTime bonusTime;
```

**特殊规则**：从 Cellxpert 迁移的历史数据 `settlementNo` 加 `CX-` 前缀区分。

### 8.2 `tb_p_c_apply_payment` · 出金申请

```java
ApplyPaymentEntity:
    private String payNo;
    private Long cpaAccount;
    private BigDecimal payAmount;       // 申请金额
    private BigDecimal realPayAmount;   // 实际支付（可能扣手续费）
    private String applyStatus;         // P（审批中）/ S（同意）/ F（拒绝）
    private LocalDateTime payTime;
```

### 8.3 `tb_p_c_account_balance` · 账户余额

```java
AccountBalanceEntity:
    private Long account;
    private BigDecimal availableBalance;  // 可用余额
    private BigDecimal lockedBalance;     // 锁定余额（待审批中）
    private String currency;
```

**关键机制**：
- 提交出金时：`availableBalance -=`，`lockedBalance +=`
- 审批通过：`lockedBalance -=`（真正出账）
- 审批拒绝：`availableBalance +=`，`lockedBalance -=`（释放回可用）

### 8.4 `tb_p_c_account_balance_record` · 余额流水

每次余额变动都生成一条流水（保留 ledger 风格的可追溯性）。

---

## 九、BPM 审批域

### 9.1 `ApproveProcessEntity` · 流程定义

```java
ApproveProcessEntity:
    private Long id;
    private String processCode;     // 流程编码（如 BONUS_APPLY）
    private String processName;
    private Integer processStatus;
    private String formSchema;      // 表单结构 JSON
    private String brand;
```

### 9.2 `ApproveNodeEntity` · 流程节点

```java
ApproveNodeEntity:
    private Long processId;
    private Integer nodeOrder;      // 节点顺序
    private String nodeName;
    private String approverType;    // ROLE / USER / EXPRESSION
    private String approverValue;   // 审批人配置（角色/账号/表达式）
    private Integer timeoutMinutes; // 超时分钟数
```

### 9.3 `ApproveWorkEntity` · 流程实例

```java
ApproveWorkEntity:
    private Long processId;
    private String processCode;
    private String businessKey;     // 业务唯一标识（如 payNo）
    private String businessValue;   // 业务值
    private String title;
    private String formData;        // 表单数据 JSON
    private Long submitter;
    private LocalDateTime submitTime;
    private Integer workStatus;     // 0=审批中 / 1=通过 / 2=拒绝 / 3=撤销 / 4=超时终止
    private LocalDateTime finishTime;
    private String brand;
```

### 9.4 `ApproveTaskEntity` · 审批任务

```java
ApproveTaskEntity:
    private Long workId;
    private Long nodeId;
    private Long approver;
    private Integer taskStatus;     // 0=待审 / 1=通过 / 2=拒绝
    private LocalDateTime approveTime;
    private String opinion;
```

### 9.5 `ApproveRecordEntity` · 审批记录

完整审批操作流水（谁在何时做了什么）。

### 9.6 启动审批的标准方式

```java
@Resource
private ApproveService approveService;

CreateApproveWorkDto req = CreateApproveWorkDto.builder()
    .processCode("BONUS_APPLY")
    .businessKey(payNo)
    .businessValue(JsonUtil.toJsonString(applyPayment))
    .title("代理 #" + account + " 出金申请 $" + amount)
    .submitter(currentUserId)
    .build();

Long workId = approveService.createApproveWork(req);
```

---

## 十、外部数据源

### 10.1 MT4 数据库

```sql
-- 用户的 MT4 账户映射
tb_account_mt4
├── user_id
├── mt4_login
└── mt4_datasource_id  -- 区分服务器

-- 交易记录
mt4_trades_modify_record
WHERE cmd < 2  -- 0=BUY / 1=SELL（其他是平仓/挂单等）
```

### 10.2 MT5 数据库

```sql
mt5_deals_modify_record
WHERE action IN (0, 1)  -- 买卖成交
```

### 10.3 DAP/User 服务（Feign）

```java
PlatformJobExternalApi.queryUserInfoList(List<Integer> userIds)
// 返回 DapUserInfoResp：姓名、邮箱、手机、国家、监管机构

PlatformAdminExternalApi.queryLeafSales(Integer userId)
// 返回销售链末级销售人员

BaseAccountMt4Api.queryIbRebateAccountListByUserId(Integer userId)
// 返回用户的 MT4 IB 返佣账户列表
```

---

## 十一、关键枚举速查

```java
// 代理类型
CpaAccountTypeEnum: CPA / IB / AFFILIATE

// 代理状态
CpaAccountStatusEnum: VALID / INVALID / SUSPENDED

// 客户状态
ClientStatusEnum: REGISTER / LIVE / QUALIFY

// 佣金模式
CommissionTypeEnum: FTD / COUNTRY / PROGRESSIVE_CPA_TIERS / DEACTIVATE_CPA

// 结算周期
SettlementCycleEnum: WEEK / BIWEEK / MONTH

// 出金状态
ApplyStatusEnum: P (Pending) / S (Success) / F (Failed)

// 审批状态
ProcessNodeStatusEnum: 待审 / 通过 / 拒绝

// 监管
LicenseEnum / RegulatorEnum

// 通知通道
NotifyChannelEnum / AlertTypeEnum
```

枚举类位置：`com.hytech.cpa.common.enums.*`

---

## 十二、DTO 与 Entity 的转换

项目使用 **MapStruct** 做 Entity ↔ DTO 转换，位于 `model/convert` 包：

```java
@Mapper(componentModel = "spring")
public interface ClientSummaryConvert {
    ClientSummaryConvert INSTANCE = Mappers.getMapper(ClientSummaryConvert.class);

    ClientSummaryDto entityToDto(ClientSummaryEntity entity);
    ClientSummaryEntity dtoToEntity(ClientSummaryDto dto);
}
```

**规则**：
- backend 层内部用 Entity
- backend → business 之间用 DTO
- business → controller 出参用 Res（或直接用 DTO）
- controller → business 入参用 Req
