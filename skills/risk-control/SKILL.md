---
name: risk-control
description: hytech-cpa 业务风控 Skill — 覆盖代理佣金管理系统的资金风险、欺诈识别、数据异常监测。包含 6 大风控场景（异常出金、刷量、虚假客户、规则套利、关系造假、规模异常）+ 实时与离线监测机制 + 应急冻结 SOP。当涉及"出金风险 / 异常佣金 / 反欺诈 / 风控规则"时调用。
version: 2.0.0
origin: SkillForge
---

# risk-control · hytech-cpa 业务风控

> 资金安全和数据正确是底线。风控不是事后追溯，而是体系化的预防、监测、响应、处置。

---

## When to Activate

- 用户说"为什么这个代理被冻结"、"这笔出金被拦截"
- 涉及"反欺诈 / 资金异常 / 数据造假 / 套利"等关键词
- 设计新功能时需要考虑风控
- 风控规则配置或调整
- 与 `oncall-trading` 配合：值班发现疑似风控问题

---

## 业务风控目标

| 目标 | 含义 |
|---|---|
| **资金安全** | 防止欺诈代理骗取佣金 / 套现 |
| **数据正确** | 防止规则配置错误、计算错误 |
| **关系真实** | 防止虚假代理关系、刷量 |
| **合规守底** | 满足监管要求（反洗钱、KYC、数据隔离） |
| **平台稳定** | 防止短时间大额异常冲击系统 |

---

## 6 大风控场景

### 场景 1 · 异常出金

**特征**：
- 单笔金额异常大（如 > $100,000）
- 短时间内多次申请
- 申请金额接近可用余额上限
- 申请后立即修改收款账户

**监测点**：
```sql
-- 单笔金额异常
SELECT * FROM tb_p_c_apply_payment
WHERE pay_amount > 100000
  AND apply_status = 'P'
  AND create_time > NOW() - INTERVAL 1 HOUR;

-- 频次异常
SELECT cpa_account, COUNT(*)
FROM tb_p_c_apply_payment
WHERE create_time > NOW() - INTERVAL 1 DAY
GROUP BY cpa_account
HAVING COUNT(*) > 5;
```

**处置 SOP**：
- 异常大单：自动转人工审批 + @ 财务 / 风控
- 高频出金：暂时锁定代理状态，调查
- 修改收款账户后立即出金：触发 24h 冷静期

### 场景 2 · 刷量（虚假交易）

**特征**：
- 代理新带来的客户大量在短时间内入金 + 立即出金
- 客户首次入金额刚好踩 FTD 阶梯触发线
- 多个客户使用相同 IP / 设备指纹
- 客户交易品种集中、手数标准化（机器人特征）

**监测点**：
```sql
-- 代理 24h 内 QFTD 触发数异常
SELECT cpa_account, COUNT(*)
FROM tb_p_bonus_qftd
WHERE bonus_time > NOW() - INTERVAL 1 DAY
GROUP BY cpa_account
HAVING COUNT(*) > 50;

-- 客户入金额集中在阶梯触发线
SELECT inviter_cpa_account, COUNT(*)
FROM tb_p_client c
JOIN tb_p_event_payment p ON c.user_id = p.client_user_id
WHERE p.amount BETWEEN 99 AND 101  -- 紧贴 $100 触发线
GROUP BY inviter_cpa_account
HAVING COUNT(*) > 10;
```

**处置 SOP**：
- 自动标记可疑 → 人工审核
- 冻结涉嫌代理的佣金生成
- 二次核验：调用 DAP/User 服务确认客户真实性
- 必要时回收已发放佣金

### 场景 3 · 虚假客户

**特征**：
- 客户姓名 / 邮箱模式化（如 user001@xxx.com, user002@xxx.com）
- 客户 KYC 资料异常
- 客户 IP 集中
- 客户开账号后短期内不交易，但满足 FTD 触发条件

**监测点**：
- 用户同步任务（`UserSyncJob`）失败率高 → 可能是假数据
- 客户邮箱域名集中
- 客户手机号归属地集中

**处置 SOP**：
- 协同 CRM / KYC 团队复核
- 暂停可疑客户的佣金生成
- 通知合规团队

### 场景 4 · 规则套利

**特征**：
- 代理在规则变更前夕大量提交 FTD
- 故意触发 ExtraBonus 边界条件
- 利用 multipleEnable 漏洞重复触发 QFTD

**监测点**：
- 规则变更前 24h 的 QFTD 触发数
- ExtraBonus 触发是否扎堆在 qualifiedMin 阈值
- 同一客户多次 QFTD 是否合理

**处置 SOP**：
- 规则变更前增加监测频率
- 规则灰度生效（部分代理先生效）
- 异常套利触发 → 人工审核后追回

### 场景 5 · 关系造假

**特征**：
- 多层级关系突然集中变更
- `path` 字段更新异常频繁
- 自循环关系（A → B → A）

**监测点**：
```sql
-- 24h 内的关系变更数
SELECT COUNT(*) FROM tb_p_cpa_account_relation
WHERE update_time > NOW() - INTERVAL 1 DAY;

-- 检测循环引用
SELECT cpa_account, path FROM tb_p_cpa_account_path
WHERE path LIKE CONCAT('%/', cpa_account, '/%/', cpa_account, '/%');
```

**处置 SOP**：
- 自动报警 → 人工核实
- 暂停可疑代理升级 / 转移
- 必要时锁定关系变更入口

### 场景 6 · 规模异常

**特征**：
- 单代理客户数突增（一夜从 10 个变 1000 个）
- 单代理佣金突增（一夜从 $100 变 $10,000）
- 单品牌出金量异常

**监测点**：
- 同比 / 环比异常（DoD / WoW）
- 偏离历史均值的 σ 倍数

**处置 SOP**：
- 自动触发"二次复核"流程
- 业务方 + 合规联合判断
- 必要时全代理停发佣金 → 全量复核

---

## 实时与离线监测机制

### 实时层（< 1 分钟）

| 触发器 | 实现 |
|---|---|
| 单笔大额出金 | Service 层加入 ApplyPaymentRiskCheck |
| 黑名单代理 | Redis 缓存（`partner:cpa:risk:blocklist`） |
| 频次限制 | RedissonRateLimiter |
| 规则套利 | EventHandle 入口加 Check |

**示例代码**：
```java
@Slf4j
@Component
@RequiredArgsConstructor
public class ApplyPaymentRiskService {

    private final RedisTemplate<String, String> redisTemplate;
    private final CommMessagePusher larkCommMessagePusher;

    public void preCheck(ApplyPaymentDto apply) {
        // 黑名单检查
        if (isBlocked(apply.getCpaAccount())) {
            throw new BizException(ClientResponseEnum.ACCOUNT_BLOCKED);
        }

        // 单笔阈值
        if (apply.getPayAmount().compareTo(new BigDecimal("100000")) > 0) {
            log.warn("large amount, account:{}, amount:{}",
                     apply.getCpaAccount(), apply.getPayAmount());
            larkCommMessagePusher.sendWarning(LarkAlertRequest.builder()
                .title("风控-大额出金")
                .description("account=" + apply.getCpaAccount())
                .build());
            // 自动转人工审批
            apply.setRequiresManualReview(true);
        }

        // 频次限制：24h 内最多 5 次
        String key = String.format("partner:cpa:apply:freq:%d",
                                    apply.getCpaAccount());
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) redisTemplate.expire(key, Duration.ofDays(1));
        if (count > 5) {
            throw new BizException(ClientResponseEnum.APPLY_FREQUENCY_LIMIT);
        }
    }
}
```

### 离线层（每日跑批）

| 任务 | 频次 | 实现 |
|---|---|---|
| 日度异常代理扫描 | 每日 02:00 | `risk-daily-scan` 任务 |
| 规则套利分析 | 每日 03:00 | 对比 24h QFTD 数据 |
| 关系造假检测 | 每日 04:00 | path 字段循环引用扫描 |
| 客户真实性核验 | 每周一 | 调 DAP 二次校验 |

---

## 应急冻结 SOP

### 当发现确凿的欺诈行为时

```
Step 1 · 立即冻结
  - 改 tb_p_cpa_account.account_status = SUSPENDED
  - 加入 Redis 黑名单
  - 该代理所有进行中的出金申请置为冻结

Step 2 · 通知相关方
  - 业务 Owner
  - 合规 / 财务
  - 客服（准备应对代理质询）

Step 3 · 调查
  - 调取该代理近 90 天活动
  - 调取关联客户活动
  - 调取关联代理（上下级）

Step 4 · 处置决策
  - 真欺诈：回收佣金、报警、加入黑名单
  - 误伤：解冻 + 道歉 + 补偿

Step 5 · 复盘
  - 风控规则是否漏报？
  - 处置流程是否高效？
  - 是否需要新增监测点？
```

### 冻结操作 SQL（必须双人复核）

```sql
-- 备份
INSERT INTO _backup_cpa_account_freeze
SELECT * FROM tb_p_cpa_account
WHERE account = ?;

-- 冻结
UPDATE tb_p_cpa_account
SET account_status = 'SUSPENDED',
    update_by = 'risk-control-team',
    update_time = NOW(),
    remark = CONCAT(IFNULL(remark,''), ' | 风控冻结: ', NOW())
WHERE account = ?;

-- 冻结进行中出金
UPDATE tb_p_c_apply_payment
SET apply_status = 'FROZEN',
    update_by = 'risk-control-team',
    update_time = NOW()
WHERE cpa_account = ? AND apply_status = 'P';
```

---

## 与外部系统的风控协作

| 团队 | 协作场景 |
|---|---|
| KYC 团队 | 客户真实性核验 |
| 合规团队 | 监管报送、AML / CTF 检查 |
| 客服团队 | 用户投诉对接 |
| 法务团队 | 重大欺诈案件 |
| 财务团队 | 资金追回 |
| 安全团队 | 黑产攻击防御 |

---

## 风控指标 Dashboard

| 指标 | 健康阈值 | 告警阈值 |
|---|---|---|
| 单日大额出金笔数 | ≤ 10 | > 30 |
| 单日代理冻结数 | ≤ 1 | > 3 |
| 单日 QFTD 触发数同比增长 | ≤ 20% | > 50% |
| 关系变更频率 | ≤ 50/day | > 200/day |
| 高风险代理占比 | ≤ 0.5% | > 2% |

监控通过 Grafana + Prometheus 实现，异常发飞书告警。

---

## 风控规则配置最佳实践

### 原则 1 · 宽进严出

入金 / 注册 / 客户绑定环节门槛适度，出金 / 佣金发放环节严控。

### 原则 2 · 分层防御

```
第一层：实时拦截（毫秒级，硬规则）
第二层：实时预警（秒级，发告警）
第三层：离线扫描（天级，发报告）
第四层：人工审核（周级，专项审查）
```

### 原则 3 · 规则可解释

- ❌ "黑盒 ML 模型直接拦截"
- ✅ 明确规则（如"单笔 > $100,000 转人工"），可追溯、可申诉

### 原则 4 · 灰度上线

新风控规则：
1. 仅告警不拦截（7 天）
2. 小批量代理拦截（7 天）
3. 全量拦截

### 原则 5 · 误伤可救济

- 必须提供申诉通道
- 误冻结后必须补偿（如手续费减免）

---

## 风控规则变更流程

1. 风控分析师 提需求 → 走 PRD 流程（`prd-template`）
2. 走可行性 + 业务评估（`feasibility-checker`）
3. 研发实现 → 审核（`code-review-rules`）
4. 测试 → 自动化覆盖（`api-test-automation`）
5. 灰度上线
6. 监控 + 复盘

---

## 与其他 Skill 协作

```
[发现异常] → 本 Skill 识别场景
        ↓
[需要看代码] → code-walkthrough Skill
[需要查业务] → hytech-cpa Skill
[需要操作飞书]→ lark-cli Skill
[严重事件升级]→ incident-response Skill
        ↓
[复盘 + 改规则] 走 PRD 流程
```

---

## 引用

- 关联 Skill：hytech-cpa, oncall-trading, incident-response, code-walkthrough, lark-cli
- 合规要求：所在地区监管法规（AU 等）
- 监管参考：AML（反洗钱）、CTF（反恐融资）
