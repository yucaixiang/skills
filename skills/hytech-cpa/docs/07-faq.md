# 07 · FAQ 与故障排查

> 本文档汇集 hytech-cpa 日常开发、值班、问答中**反复出现的问题**及**标准排查路径**。
> 遇到问题先查这里，找不到再问人。

---

## 一、业务概念类

### Q1：QFTD 是什么？
**Qualified First Time Deposit**。客户的首次入金满足一定条件（金额 / 时间窗 / 交易量）才"合格"，触发佣金。
具体条件配置在 `RuleQftdConditionEntity`。

### Q2：FTD、Country、Progressive 有什么区别？
- **FTD**：按客户**首次入金额**分阶梯（如 $100-$500 给 $20）
- **Country**：按客户**所在国家**给不同金额
- **Progressive**：按代理**累计客户/入金/交易量**分阶段（越多奖励越多）

代码层面：`BonusCalculator` 接口 + 3 个实现类。

### Q3：CPA / IB / AFFILIATE 三种代理类型的区别？
- **CPA**：Cost Per Action，按客户行为计费的代理
- **IB**：Introducing Broker，介绍经纪人，传统代理
- **AFFILIATE**：联盟营销代理（更偏线上引流）

枚举：`CpaAccountTypeEnum`。三者业务流程基本一致，差异在佣金规则和监管要求。

### Q4：什么是 ruleSnapshot？为什么要有？
代理签署协议时，把**当前规则的完整内容**冻结成 JSON 存进 `AgreementEntity.ruleSnapshot`。
即使规则后续修改，已签协议仍按快照计算佣金，**避免事后改规则坑代理**。

### Q5：path 字段是干什么的？
`tb_p_cpa_account_path.path` 存代理的完整上下级路径（如 `/100/200/300/`），用于快速查询某主账户下所有下级（`LIKE '/100/%'`），避免递归查询关系表。

---

## 二、佣金计算类

### Q6：客户达成 QFTD 了，为什么没生成佣金？

**排查步骤**：
1. 查 `tb_p_client.qualifyTime` 是否有值
2. 查 `tb_p_bonus_qftd` 有没有对应记录
3. 如果都有但没结算单：
   - 查 `tb_p_agreement` 看是否已签（status=S）
   - 查 `tb_p_rule.rangeDateStart/End` 看是否在规则有效期内
   - 查 `tb_p_rule_cpa` 看代理是否关联到规则
4. 查 ERROR 日志，按 `eventid` 看完整链路
5. 看飞书告警「佣金计算」群有没有相关 title

### Q7：FTD 命中多档怎么办？
告警：`title="佣金计算"`，desc 包含 "qualified multiple bonus items"。
**根因**：`RuleQftdBonusItem` 区间配置重叠（如 `[100, 500)` 和 `[400, 1000)`）。
**修复**：去后台 `BsnRuleGroupController` 改 Tier 配置，让区间不重叠。

### Q8：客户多次入金累计佣金不对？
检查规则的 `multipleEnable` 字段：
- `false`：只计算首次 QFTD，后续入金不计
- `true`：每次入金都可能触发新的 QFTD

Progressive 模式下，看 `ClientSummaryDto.cumulativeFtdAmount` 是否正确累加。

### Q9：分润金额对不上？

**排查步骤**：
1. 查 `tb_p_cpa_account_path.path` 是否准确反映关系链
2. 查 `tb_p_rule_bonus_share` 是否有规则
3. 查 `tb_p_rule_bonus_share_item` 的 `childLevel`、`shareRate`
4. 计算：原始佣金 × shareRate = 分润金额
5. 查 `tb_p_bonus_share` 是否真的写了对应记录

### Q10：为什么有的代理拿不到分润？
- 该代理的某一级**没有配置分润规则**
- `path` 字段未及时更新（关系变更没同步）
- 分润计算异步触发失败（看 ERROR 日志）

---

## 三、结算支付类

### Q11：代理可用余额对不上？
对账步骤：
```sql
-- 1. 看流水
SELECT * FROM tb_p_c_account_balance_record
WHERE account = ? ORDER BY create_time;

-- 2. 看余额表
SELECT * FROM tb_p_c_account_balance WHERE account = ?;

-- 3. 流水累计应该等于余额表的 available + locked
```
如不一致，说明有变动没走流水。

### Q12：代理申请出金后金额没扣？
排查：
1. 看 `tb_p_c_apply_payment` 状态：
   - `P`：审批中，`available` 已减、`locked` 已加 ✅
   - `S`：通过，`locked` 应释放出账 ✅
   - `F`：拒绝，`locked` 应释放回 `available` ✅
2. 看 `AccountBalanceCoreService.changeBalance(...)` 的事务有没有报错

### Q13：审批通过了但代理没到账？
- 查 `ApproveWorkEntity.workStatus = 1`（通过）
- 查 `ApproveWorkFinishedHandler` 有没有正确触发（看 `BonusApproveFinishedHandler` 的日志）
- 查支付网关调用记录（外部依赖）

### Q14：结算单状态一直没变？
- 查 `SettlementEntity.autoFlag`：1=自动 / 0=手动
- 自动结算靠 `cpa-job-executor/payment/AccountBalanceJob` 跑批
- 手动结算需要管理员在后台触发

---

## 四、审批 BPM 类

### Q15：审批一直卡着没人审？
1. 查 `ApproveTaskEntity` 是否有 `taskStatus = 0` 的待审任务
2. 查任务的 `approver` 是谁
3. 看 `ApproveNodeEntity.approverType`：
   - `USER`：直接看 `approverValue` 是哪个用户
   - `ROLE`：看哪些用户拥有这个角色
   - `EXPRESSION`：看表达式动态算出谁
4. 提醒审批人去后台处理

### Q16：审批超时怎么办？
- 超时分钟数配置在 `ApproveNodeEntity.timeoutMinutes`
- 定时任务扫描超时项，状态置为 4（超时终止）
- 业务侧 `ApproveWorkFinishedHandler` 收到 finalStatus=4，需要补偿处理（如释放锁定余额）

### Q17：怎么撤回已提交的审批？
- 调 `approveService.cancel(workId, userId, userName)`
- 仅提交人本人可撤
- 必须在 `workStatus = 0`（审批中）状态下

---

## 五、数据同步类

### Q18：用户信息和外部 DAP 不一致？
- 查 `UserSyncJob` 最近一次执行日志（XxlJob 后台）
- 看 `UserSyncCoreServiceImpl.sync()` 的 ERROR 日志
- 飞书告警「用户同步失败」群

### Q19：MT4/MT5 交易数据没拉到？
- 查多数据源配置是否正确（`mt4_datasource_id`）
- 查 `tb_account_mt4` 的映射是否完整
- 查 `BatchJob` 的扫表时间窗设置

### Q20：从 Cellxpert 迁移的历史数据怎么识别？
- `SettlementEntity.settlementNo` 以 `CX-` 开头
- 查询时可加条件 `settlementNo NOT LIKE 'CX-%'` 排除历史

---

## 六、技术问题类

### Q21：Redis 锁拿不到怎么办？
```java
boolean locked = redissonLock.lock(lockKey, 30, TimeUnit.SECONDS);
if (!locked) {
    // 选项 1：抛异常，让上层重试
    throw new BizException(JobResponseEnum.LOCK_FAILED);
    // 选项 2：降级处理（仅适合非关键操作）
    return defaultResult();
}
```
**禁止**：拿不到锁就跳过关键业务逻辑。

### Q22：Kafka 消息丢了？
- 生产端：开 `acks=all`，必返回成功才算发送
- 生产端：失败重试 + 死信队列 + 飞书告警
- 消费端：手动 ACK，业务成功才提交 offset
- 兜底：业务表加 `sync_status` 字段，补偿 Job 定时扫未同步的

### Q23：定时任务突然不跑了？
1. 查 XxlJob 后台任务是否被禁用
2. 查执行器是否在线
3. 看任务最后一次执行日志
4. 看 `cpa-job-executor` 服务是否正常

### Q24：Feign 调用一直超时？
- 检查目标服务是否健康（Consul 看注册状态）
- 检查超时配置（`feign.client.config.default.readTimeout`）
- 检查目标接口是否变慢（看监控）
- 临时方案：客户端加重试（Resilience4j）

### Q25：QlExpress 规则执行报错？
- 看表达式语法（QlExpress 类 JavaScript 语法）
- 看上下文变量是否都 put 进去了
- 看变量类型是否匹配（BigDecimal vs Double）
- Debug：开 `expressRunner.execute(... , true, true)` 第二个 true 是 trace

---

## 七、多品牌 / 多环境类

### Q26：跨品牌数据混了？
- 检查 SQL 是否带 `WHERE brand = ?`
- 检查 Entity 是否正确赋值 `brand` 字段
- 检查 Consul 服务发现 `tag` 是否正确

### Q27：dev 环境调用了 prod 服务？
- 检查启动参数 `-Dconsul.host` / `-Dconsul.port`
- 检查 `consul.tags` / `consul.select.tag`
- 必要时检查 `/etc/hosts`

---

## 八、监控告警类

### Q28：飞书告警一直不响？
排查告警链路：
1. 查 ERROR 日志是否真的写了
2. 查 `larkCommMessagePusher.sendAlert(...)` 是否被调用
3. 查 `cpa.alert.type = lark` 配置
4. 查 `cpa.lark.appId/appSecret/chatId` 配置
5. 查多维表格责任人配置是否对（title 匹配）
6. 查 Lark API 返回（看本地日志）

### Q29：告警自动 @ 人不准？
- 责任人配置在多维表格，按 `LarkAlertRequest.title` 匹配
- 检查多维表格中 title 字段拼写
- 看 Bitable 缓存 TTL（默认 5 分钟），更新后等等

### Q30：怎么静默某个告警？
- 短期：多维表格里把对应 title 的 people 置空
- 长期：业务代码里加判断不发送，或下调级别（sendAlert → sendInfo）

---

## 九、性能问题

### Q31：报表查询慢？
- 走只读服务 `cpa-client-read-service`
- 大查询用 `ClientSummaryEntity` 等聚合表
- 必要时加分页 + 索引

### Q32：批量任务 OOM？
- 用游标分页 / 分段处理
- 不要一次 `selectAll` 几十万行
- EasyExcel 导出用 stream API

### Q33：佣金计算慢？
- Redisson 锁粒度：按 `clientUserId` 而不是全局锁
- 规则缓存（Redis）
- 异步触发分润计算（runAsync）
- 批量处理用 Stream 流而不是嵌套循环

---

## 十、开发流程类

### Q34：要加一个新的佣金模式怎么办？

**3 步走**：
1. 加枚举：`CommissionTypeEnum` 添加新值
2. 新增 Calculator：实现 `BonusCalculator` 接口（getType + calculate）
3. 加 `@Component` Spring 会自动注册到 `bonusCalculatorMap`

**不要**：动 `BonusCalculationCoreServiceImpl` 主流程。

### Q35：要发布给跨域服务调用怎么办？

**步骤**：
1. 在自己域的 `*-platform-api` 模块定义 Feign 接口
2. 在 backend 提供 `/bck/*` 接口实现
3. 提供数据契约 DTO（不暴露 Entity）
4. 文档化（Knife4j 注解）

**不要**：让对方直接调你的 `*-internal-api`（仅限本域用）。

### Q36：要新增一张表怎么办？
1. 建表语句走 DBA 流程
2. 生成 `XxxEntity`（用代码生成器或手写）
3. 创建 `XxxMapper` + `XxxMapper.xml`
4. 创建 `XxxRepository` 封装常用查询
5. **不要**：在 Service 里直接调 Mapper

### Q37：要修复一个老 bug，但代码风格不符合规范怎么办？
**外科手术原则**：
- 只改 bug 直接相关的代码
- 不要"顺手"重构周边代码（即使看起来很丑）
- 如果发现严重违规，提单子让 owner 决定

---

## 十一、参考资料

- README.md
- 数据迁移doc/hytech-cpa项目理解.md
- docs/hytech-cpa-alert使用说明.md
- 飞书告警自动@负责人完整实施方案.md
- 多维表格查询功能使用说明.md
