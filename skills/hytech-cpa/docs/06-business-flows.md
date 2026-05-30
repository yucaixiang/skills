# 06 · 核心业务流程

> 本文档讲透 hytech-cpa 几个关键业务流程的端到端链路：佣金计算、多层级分润、结算支付、BPM 审批、用户同步。
> 每个流程附带：触发入口 → 串联 Service → 落库实体 → 异常分支。
> 排查任何业务问题（"为什么佣金没生成"、"分润金额不对"等）前，先在本文档找对应流程。

---

## 流程 1 · 佣金计算（最核心）

### 1.1 触发入口

佣金计算由"客户事件"驱动。事件来源有 4 个：

| 事件来源 | 触发方式 | 入口 |
|---|---|---|
| **MT4/MT5 入金** | 定时任务扫表 | `cpa-job-executor/bonus/BatchJob` |
| **MT4/MT5 交易** | 定时任务扫表 | 同上 |
| **手动补偿** | 后台触发 | `cpa-job-executor/bonus/CompensationJob` |
| **批量校验** | 定时任务 | `cpa-job-executor/bonus/VerifyJob` |

所有路径最终都会进入：

```java
EventHandleCoreService.handle(List<EventDto> events)
```

### 1.2 完整流程链路

```
┌──────────────────────────────────────────────────────────────────────┐
│  Step 1 · 事件采集（cpa-job-executor）                                │
│  BatchJob → 扫描 tb_payment_deposit / mt4_trades_modify_record       │
│           → 转换为 EventDto                                          │
│           → 调用 EventHandleCoreService.handle(events)               │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 2 · 事件分发（EventHandleCoreServiceImpl）                      │
│                                                                      │
│  MDC.put("eventid", UUID.randomUUID().toString())  // 链路追踪        │
│                                                                      │
│  events.stream()                                                     │
│    .sorted(by clientUserId, eventTime)  // 同客户事件按时间有序处理   │
│    .forEach(event -> handle(event))                                  │
│                                                                      │
│  for each event:                                                     │
│    redissonLock.lock("BONUS_LOCK::" + clientUserId)  // 客户维度加锁  │
│      → loadClientSummary(clientUserId) → ClientSummaryDto            │
│      → switch(event.type):                                           │
│           case PAYMENT → handlePayment(event, clientSummary)         │
│           case TRADE   → handleTrade(event, clientSummary)           │
│    redissonLock.unlock(...)                                          │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 3 · QFTD 条件判断（ConditionCoreService）                       │
│                                                                      │
│  ConditionCoreService.qftdQualified(clientSummary):                  │
│    检查规则中的 QFTD 条件：                                          │
│      ✓ 入金额度 ≥ 阈值                                               │
│      ✓ 开账号天数 ≤ 时间窗                                           │
│      ✓ 交易量 ≥ 最小手数                                              │
│      ✓ ...                                                           │
│    → 通过：clientSummary.qftdQualified = true                        │
│      → 写 tb_p_bonus_qftd                                            │
│    → 失败：return（不触发佣金）                                       │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 4 · 加载规则（BaseCalculationCoreService.loadRule）             │
│                                                                      │
│  loadRule(clientSummary):                                            │
│    根据 cpaAccount + clientCountry 找到对应的 RuleDto                │
│    优先级：CPA 专属规则 > 规则组 > 默认规则                          │
│    加载 AgreementEntity.ruleSnapshot  ⚡ 用快照而非最新规则           │
│    设置：clientSummary.rule, clientSummary.ruleQftdBonusItems        │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 5 · 佣金计算调度（BonusCalculationCoreServiceImpl）             │
│                                                                      │
│  calculateQftdBonus(clientSummary):                                  │
│    if (rule.isDeactivate()) return                                   │
│    if (clientSummary.isInvalidMultipleRule()) return  // 多次 QFTD   │
│                                                                      │
│    BonusCalculator calculator =                                      │
│        bonusCalculatorMap.get(rule.commissionType)                   │
│    calculator.calculate(clientSummary)  // 策略模式分发              │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 6 · 具体计算（4 种 BonusCalculator）                            │
│                                                                      │
│  FTDBonusCalculator:                                                 │
│    在 RuleQftdBonusItems 中找 ftdAmount ∈ [from, to) 的档位          │
│    if (hitItems.size() > 1) → 发飞书告警「规则错误」                  │
│    if (hitItems.isEmpty())  → return                                  │
│    → clientSummary.doCompletedBonus(bonusQftdDto, true)              │
│                                                                      │
│  CountryBonusCalculator:                                             │
│    按 client.country 找 TierCountryEntity → 取对应金额                │
│                                                                      │
│  ProgressiveBonusCalculator:                                         │
│    多次入金累计 → 找当前阶段 Tier → 计算增量                          │
│                                                                      │
│  DeactivateBonusCalculator:                                          │
│    （特殊处理，停用代理的尾期处理）                                  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 7 · 落库（BonusTransactional 包事务）                           │
│                                                                      │
│  bonusTransactional.completeBonus(bonusQftdDto):                     │
│    INSERT tb_p_bonus_qftd          // QFTD 记录                       │
│    INSERT tb_p_settlement          // 结算单（自动生成）              │
│    UPDATE tb_p_client_summary      // 更新客户汇总                    │
│    KafkaTemplate.send(BONUS_TOPIC, bonusMessage)  // 通知下游         │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼ (异步)
┌──────────────────────────────────────────────────────────────────────┐
│  Step 8 · 分润计算（异步触发）                                        │
│                                                                      │
│  runAsync(() -> bonusShareCoreService.calculateShare(bonusQftd))     │
│  → 详见「流程 2 · 多层级分润」                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.3 涉及的核心类

| 类 | 位置 | 职责 |
|---|---|---|
| `EventHandleCoreServiceImpl` | core-backend | 事件入口 + 编排 |
| `BonusCalculationCoreServiceImpl` | core-backend | 策略分发 |
| `ConditionCoreService` | core-backend | QFTD/Extra 条件判断 |
| `FTDBonusCalculator` | core-backend | FTD 模式 |
| `CountryBonusCalculator` | core-backend | Country 模式 |
| `ProgressiveBonusCalculator` | core-backend | Progressive 模式 |
| `BonusTransactional` | core-backend | 事务包装 |
| `RedissonLock` | common | 分布式锁 |
| `CommMessagePusher` | alert | 异常告警 |
| `KafkaTemplate` | spring-kafka | 事件下游通知 |

### 1.4 常见异常分支

| 异常 | 根因 | 处理 |
|---|---|---|
| `RULE_NOT_EXISTS` | 没找到匹配规则 | 抛 BizException，记日志，告警 |
| FTD 命中多条规则 | 规则配置错误（区间重叠） | 不计算，发飞书告警「规则错误」 |
| ruleQftdBonusItems is null | 规则数据被删 | 跳过该客户，记 error log |
| Redis 锁获取失败 | 并发激烈 | 抛异常，由外层重试 |
| Kafka 发送失败 | MQ 故障 | 落到本地补偿队列，定时重试 |

### 1.5 排查工具

- **MDC eventid**：每次事件处理生成 UUID，日志聚合按 eventid 查整条链路
- **飞书告警**：`title="佣金计算"`、`title="规则错误"` 的告警自动 @ 责任人
- **手动补偿**：`CompensationJob` 可指定 clientUserId 重新计算
- **校验任务**：`VerifyJob` 每天扫描，发现"应有佣金却没有"的客户

---

## 流程 2 · 多层级分润

### 2.1 触发入口

佣金计算 Step 7 落库成功后，**异步** 触发分润：

```java
runAsync(() -> bonusShareCoreService.calculateShare(bonusQftdDto));
```

### 2.2 完整流程链路

```
┌──────────────────────────────────────────────────────────────────────┐
│  Step 1 · 查找上级链路                                                │
│                                                                      │
│  BonusShareCoreService.calculateShare(bonusQftd):                    │
│    Long childAccount = bonusQftd.cpaAccount  // 产生佣金的代理        │
│    BigDecimal childBonus = bonusQftd.bonusAmount                     │
│                                                                      │
│    // 通过 path 字段一次查出所有上级                                  │
│    String path = cpaAccountPathRepository                            │
│        .findByCpaAccount(childAccount).path                          │
│        // 如：/master_id/level2_id/level3_id/                         │
│    List<Long> ancestors = parsePath(path)  // [master, level2, level3]│
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 2 · 加载分润规则                                                │
│                                                                      │
│  for each ancestor in ancestors:                                     │
│    RuleBonusShareEntity rule =                                       │
│        ruleBonusShareRepository.findByCpaAccount(masterAccount)      │
│    if (rule == null) skip                                            │
│                                                                      │
│    List<RuleBonusShareItemEntity> items =                            │
│        ruleBonusShareItemRepository.findByRuleId(rule.id)            │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 3 · 逐级计算分润                                                │
│                                                                      │
│  for each ancestor in ancestors:                                     │
│    int level = calcLevel(ancestor, childAccount)  // 隔几级           │
│                                                                      │
│    RuleBonusShareItemEntity item = items.stream()                    │
│        .filter(i -> i.childLevel == level)                           │
│        .findFirst()                                                   │
│        .orElse(null)                                                  │
│                                                                      │
│    if (item == null) continue                                        │
│                                                                      │
│    BigDecimal shareAmount = childBonus.multiply(item.shareRate)      │
│                                                                      │
│    BonusShareEntity share = BonusShareEntity.builder()               │
│        .masterAccount(masterAccount)                                 │
│        .parentAccount(ancestor)                                      │
│        .childAccount(childAccount)                                   │
│        .bonusAmount(childBonus)                                      │
│        .shareAmount(shareAmount)                                     │
│        .bonusDate(LocalDate.now())                                   │
│        .build()                                                       │
│    bonusShareRepository.insert(share)                                │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Step 4 · 异步触发结算单生成                                          │
│                                                                      │
│  for each ancestor with share > 0:                                   │
│    INSERT tb_p_c_settlement (settlementType = "Bonus")               │
│      cpaAccount = ancestor                                           │
│      settlementAmount = shareAmount                                  │
│      autoFlag = 1                                                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.3 数值示例

```
关系链：A(顶级) → C(level1) → B(level2)
分润规则（A 配置）：level1 = 10%, level2 = 5%

B 的客户产生 $100 佣金：
  → BonusShareEntity:
      masterAccount=A, parentAccount=C, childAccount=B, share=$5  (B→C，5%)
  → BonusShareEntity:
      masterAccount=A, parentAccount=A, childAccount=B, share=$10 (B→A，10%)

最终：
  B 入账 $100
  C 入账 $5
  A 入账 $10
```

### 2.4 注意事项

- ⚡ **path 字段必须准确**：关系变更（如转移代理）务必同步更新 `tb_p_cpa_account_path`
- ⚡ **小心循环引用**：path 解析时检测循环（A→B→A）
- ⚡ **分润不重叠**：同一笔佣金对同一上级最多产生 1 条分润记录
- 日度汇总在凌晨跑批：`BonusShareDayJob`

---

## 流程 3 · 结算与出金支付

### 3.1 流程图

```
┌─────────────┐    定时/事件     ┌──────────────────────┐
│ 佣金记录    │ ──────────────→ │ tb_p_c_settlement     │
│ 分润记录    │   生成结算单     │ 结算单                │
└─────────────┘                 │ status: 待出金        │
                                └──────────────────────┘
                                          │
                                          ▼ 代理查看
                                ┌──────────────────────┐
                                │ AccountBalanceEntity │
                                │ availableBalance     │
                                └──────────────────────┘
                                          │
                                          ▼ 代理申请出金
                                ┌──────────────────────┐
                                │ ApplyPaymentEntity   │
                                │ applyStatus = P      │
                                │                      │
                                │ 同时：                │
                                │   available -=       │
                                │   locked +=          │
                                └──────────────────────┘
                                          │
                                          ▼ 启动 BPM
                                ┌──────────────────────┐
                                │ ApproveWorkEntity    │
                                │ processCode=BONUS_APPLY│
                                │ businessKey=payNo     │
                                └──────────────────────┘
                                          │
                                  ┌───────┼───────┐
                                  ▼               ▼
                            ┌─────────┐     ┌─────────┐
                            │ 审批通过 │     │ 审批拒绝 │
                            └────┬────┘     └────┬────┘
                                 │               │
                                 ▼               ▼
                       ┌──────────────┐  ┌──────────────┐
                       │ applyStatus  │  │ applyStatus  │
                       │ = S          │  │ = F          │
                       │              │  │              │
                       │ locked -=    │  │ available += │
                       │ (真正出账)   │  │ locked -=    │
                       │              │  │ (释放回可用) │
                       │ 调用支付网关 │  └──────────────┘
                       └──────────────┘
```

### 3.2 核心代码位置

| 步骤 | 位置 |
|---|---|
| 自动结算单生成 | `cpa-job-executor/payment/AccountBalanceJob` |
| 代理查看余额 | `BsnSettlementController#queryBalance` (client) |
| 代理申请出金 | `BsnSettlementController#applyPayment` (client) |
| 后台审批 | `BsnBonusApproveController` (admin) |
| 余额变动 | `AccountBalanceCoreService.changeBalance(...)` |

### 3.3 注意事项

- ⚡ **数据一致性**：余额变动必须在事务内，同时更新 `AccountBalanceEntity` 和 `AccountBalanceRecordEntity`
- ⚡ **锁定余额机制**：避免代理同时提交多笔超额申请
- 结算单号迁移规则：Cellxpert 历史数据 `settlementNo` 加 `CX-` 前缀
- 退款场景：退款（settlementType=Deduction）会产生负数结算单

---

## 流程 4 · BPM 审批

### 4.1 启动审批的标准代码

任何业务想发起审批：

```java
@Resource
private ApproveService approveService;

public void submitForApproval(ApplyPaymentEntity apply) {
    CreateApproveWorkDto req = CreateApproveWorkDto.builder()
        .processCode("BONUS_APPLY")              // 流程编码
        .businessKey(apply.getPayNo())            // 业务唯一 key
        .businessValue(JsonUtil.toJsonString(apply))  // 业务内容
        .title("代理 #" + apply.getCpaAccount()
             + " 出金申请 $" + apply.getPayAmount())
        .submitter(currentUserId())
        .build();

    Long workId = approveService.createApproveWork(req);
    apply.setApproveWorkId(workId);
    apply.setApplyStatus(ApplyStatusEnum.P.getCode());
    applyPaymentRepository.update(apply);
}
```

### 4.2 审批操作

```java
// 同意
approveService.passed(taskId, userId, userName, opinion);

// 拒绝
approveService.rejected(taskId, userId, userName, opinion);

// 撤销
approveService.cancel(workId, userId, userName);

// 通用入口
approveService.approve(workId, taskId, userId, userName, opinion, action);
```

### 4.3 流程完成回调

业务方实现 `ApproveWorkFinishedHandler` SPI 接口：

```java
@Component
public class BonusApproveFinishedHandler implements ApproveWorkFinishedHandler {

    @Override
    public String supportProcessCode() { return "BONUS_APPLY"; }

    @Override
    public void onFinished(ApproveWorkEntity work, Integer finalStatus) {
        // work.businessKey 是 payNo
        ApplyPaymentEntity apply = applyPaymentRepository
            .findByPayNo(work.getBusinessKey());

        if (finalStatus == 1) {  // 通过
            apply.setApplyStatus(ApplyStatusEnum.S.getCode());
            // 触发支付
            paymentService.executePay(apply);
        } else {  // 拒绝
            apply.setApplyStatus(ApplyStatusEnum.F.getCode());
            // 释放锁定余额
            balanceService.releaseLocked(apply.getCpaAccount(),
                                         apply.getPayAmount());
        }
        applyPaymentRepository.update(apply);
    }
}
```

### 4.4 超时机制

`ApproveNodeEntity.timeoutMinutes` 配置超时分钟数，定时任务扫描 `tb_approve_work` 中 `workStatus=0` 且 `submitTime + timeout < now` 的工作项，标记为 `4`（超时终止）。

### 4.5 状态机

```
ApproveWorkEntity.workStatus:

  0 审批中
   ├─ → 1 通过（所有节点通过）
   ├─ → 2 拒绝（任一节点拒绝）
   ├─ → 3 撤销（提交人撤回）
   └─ → 4 超时终止（节点超时）
```

---

## 流程 5 · 用户信息同步

### 5.1 流程图

```
┌─────────────────────────────────┐
│  XxlJob: UserSyncJob            │
│  每 N 分钟执行                  │
└─────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────┐
│  UserSyncCoreServiceImpl.sync() │
└─────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Step 1 · 查本地需同步的用户                            │
│    SELECT user_id FROM tb_p_cpa_user                    │
│    SELECT user_id FROM tb_p_client                      │
└─────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Step 2 · Feign 调用 DAP/User 服务                       │
│    PlatformJobExternalApi.queryUserInfoList(userIds)    │
│      → List<DapUserInfoResp>                            │
│        - userId, name, email, mobile                    │
│        - country, regulator, license                    │
└─────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Step 3 · 比对差异                                       │
│    for each dapUser in dapUserInfoList:                 │
│      localUser = localMap.get(dapUser.userId)           │
│      if (dapUser.email != localUser.email               │
│          || dapUser.mobile != localUser.mobile          │
│          || ...) {                                       │
│        diffs.add(dapUser)                               │
│      }                                                  │
└─────────────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Step 4 · 批量更新本地                                   │
│    cpaUserRepository.batchUpdate(diffs)                 │
│    clientRepository.batchUpdate(diffs)                  │
│                                                         │
│    KafkaTemplate.send(USER_SYNC_TOPIC, message)         │
│    （通知下游可能依赖用户信息的服务）                   │
└─────────────────────────────────────────────────────────┘
```

### 5.2 注意事项

- 同步任务的"批量大小"配置在 `JobConstant`
- 用户不存在 / 已注销的情况：标记 `tb_p_cpa_user.status = INVALID`
- 同步失败重试：XxlJob 自带重试机制 + 飞书告警

---

## 流程 6 · Cellxpert 历史数据迁移（背景知识）

> 迁移已基本完成，但仍然影响系统行为，开发新功能时需要知道这些约定。

### 6.1 迁移数据标记

```
SettlementEntity.settlementNo 前缀：
  CX-XXX = 从 Cellxpert 迁移的历史结算单
  无前缀 = hytech-cpa 自己生成
```

### 6.2 迁移期定时同步任务

历史上有 5 个定时任务把公司数据同步给 Cellxpert（现已停用）：

```
公司业务数据库
  tb_user + tb_user_extends（cxd 字段）
  tb_payment_deposit
  tb_payment_withdraw
  mt4_trades_modify_record
  mt5_deals_modify_record
        ↓ 5 个 CellxpertTasks（每分钟）
  tb_cellxpert_registration
  tb_cellxpert_transaction
  tb_cellxpert_position
        ↓ Cellxpert 拉取
  Cellxpert
        ↓ 佣金计算后回传
  公司其他项目（佣金报告数据）
```

代码位置（已淘汰）：`marketo-cron/src/main/java/com/ty/task/CellxpertTasks.java`

### 6.3 详细迁移文档

- `数据迁移doc/hytech-cpa项目理解.md` · 完整业务理解
- `数据迁移doc/数据迁移总计划.md`
- `数据迁移doc/佣金迁移专项方案.md`
- `数据迁移doc/迁移设计流程图-汇报版.md`

---

## 排查问题的标准动作

按问题类型对照流程：

| 问题 | 看哪个流程 | 重点检查 |
|---|---|---|
| 客户达成 QFTD 却没生成佣金 | 流程 1 | Step 3 QFTD 条件 + Step 4 规则加载（协议是否签？快照对吗？） |
| 佣金金额不对 | 流程 1 Step 6 | Tier 配置 + ftdAmount 落在哪个区间 |
| FTD 命中多档 | 流程 1 异常分支 | 规则配置区间重叠 → 检查 RuleQftdBonusItem |
| 分润金额不对 | 流程 2 Step 3 | shareRate + childLevel 配置 |
| 上级没拿到分润 | 流程 2 Step 1 | `tb_p_cpa_account_path.path` 是否准确 |
| 出金申请通过了但没到账 | 流程 3 + 4 | ApproveWorkFinishedHandler 是否正确触发 |
| 余额不一致 | 流程 3 | `tb_p_c_account_balance_record` 流水是否完整 |
| 审批一直卡着 | 流程 4 | 检查 `ApproveTaskEntity` 是否有待审任务 |
| 用户信息陈旧 | 流程 5 | `UserSyncJob` 是否正常 |
