---
name: user-analysis
description: 用户分析框架 Skill — 针对 hytech-cpa 的代理（IB/CPA/AFFILIATE）与客户（Forex 终端用户）做画像、分层、留存、流失、价值贡献等系统化分析。提供 RFM、AARRR、漏斗、Cohort、LTV 五大经典模型在 hytech-cpa 场景下的具体落地 SQL。当用户说"分析一下 xx 用户 / xx 代理 / xx 群体"时调用。
version: 1.0.0
origin: SkillForge
---

# user-analysis · 用户分析框架

> 让运营 / 产品 / BD 用经典分析模型理解代理和客户，而不是凭直觉拍。

---

## When to Activate

- 用户说"分析 xx 代理群体"、"看看哪些客户最有价值"
- 用户说"为什么 xx 用户流失了"、"留存怎么样"
- 用户说"做个用户画像 / 分层"
- 与 `data-query` 串联：先查数据，再分析

---

## 5 大经典模型在 hytech-cpa 场景下的落地

| 模型 | 适用场景 |
|---|---|
| **RFM** | 代理价值分层（最近/频次/金额） |
| **AARRR** | 代理生命周期阶段分析（获取/激活/留存/收益/推荐） |
| **漏斗** | 客户转化路径分析（注册→开账号→QFTD） |
| **Cohort** | 同期群留存分析 |
| **LTV** | 代理终身价值预测 |

---

## 模型 1 · RFM 代理价值分层

### 概念

| 维度 | hytech-cpa 含义 | 字段来源 |
|---|---|---|
| **R**ecency | 最近一次产生佣金距今天数 | `tb_p_c_settlement.bonus_time` |
| **F**requency | 90 天内活跃天数 | `tb_p_bonus_share_day` |
| **M**onetary | 90 天累计佣金金额 | `tb_p_c_settlement` |

### 分层 SQL

```sql
WITH rfm AS (
  SELECT
    cpa_account,
    DATEDIFF(CURDATE(), MAX(bonus_time)) AS recency_days,
    COUNT(DISTINCT DATE(bonus_time))      AS frequency_days,
    SUM(settlement_amount)                AS monetary
  FROM tb_p_c_settlement
  WHERE bonus_time > NOW() - INTERVAL 90 DAY
    AND brand = 'AU'
  GROUP BY cpa_account
),
scored AS (
  SELECT
    cpa_account,
    NTILE(5) OVER (ORDER BY recency_days ASC)  AS r_score,  -- 越小越好
    NTILE(5) OVER (ORDER BY frequency_days DESC) AS f_score,
    NTILE(5) OVER (ORDER BY monetary DESC)       AS m_score
  FROM rfm
)
SELECT
  cpa_account,
  r_score, f_score, m_score,
  CASE
    WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN '⭐ 核心代理'
    WHEN r_score >= 4 AND f_score >= 4 AND m_score <= 2 THEN '💎 潜力新星'
    WHEN r_score >= 4 AND m_score >= 4 THEN '🔥 高价值待挽留'
    WHEN r_score <= 2 AND f_score >= 4 THEN '⚠️ 即将流失'
    WHEN r_score <= 2 AND f_score <= 2 THEN '😴 已流失'
    ELSE '🟢 普通'
  END AS segment
FROM scored;
```

### 分层后的运营策略

| 群体 | 占比预期 | 策略 |
|---|---|---|
| ⭐ 核心代理 | 5-10% | VIP 服务、独家活动、保留 |
| 💎 潜力新星 | 10-15% | 培养、加大投入、培训 |
| 🔥 高价值待挽留 | 5-10% | 主动联系、专属优惠 |
| ⚠️ 即将流失 | 10-20% | 召回活动、原因调研 |
| 😴 已流失 | 20-30% | 唤醒 or 放手 |
| 🟢 普通 | 30-50% | 自然运营 |

---

## 模型 2 · AARRR 代理生命周期

### 5 阶段

```
Acquisition (获取)   Activation (激活)   Retention (留存)
                                                ↓
                       Revenue (收益)  ← Referral (推荐)
```

### 在 hytech-cpa 中的具体指标

| 阶段 | 指标 | 计算方式 |
|---|---|---|
| **Acquisition** | 月新增代理数 | COUNT(tb_p_cpa_account WHERE register_time IN month) |
| **Activation** | 30 天内完成首笔客户绑定 | tb_p_account_client_relation 首条 bind_time |
| **Retention** | 30/60/90 天活跃 | tb_p_bonus_share_day 有记录 |
| **Revenue** | 30 天累计佣金 | tb_p_c_settlement SUM(amount) |
| **Referral** | 30 天内邀请的下级代理数 | tb_p_cpa_account_relation parent=self |

### 5 阶段漏斗 SQL

```sql
WITH cohort AS (
  SELECT account, register_time
  FROM tb_p_cpa_account
  WHERE register_time >= '2026-04-01' AND register_time < '2026-05-01'
    AND brand = 'AU'
)
SELECT
  COUNT(*) AS acquired,

  COUNT(CASE WHEN EXISTS (
    SELECT 1 FROM tb_p_account_client_relation acr
    WHERE acr.parent_cpa_account = c.account
      AND acr.bind_time < c.register_time + INTERVAL 30 DAY
  ) THEN 1 END) AS activated,

  COUNT(CASE WHEN EXISTS (
    SELECT 1 FROM tb_p_bonus_share_day b
    WHERE b.cpa_account = c.account
      AND b.bonus_date BETWEEN c.register_time + INTERVAL 30 DAY
                            AND c.register_time + INTERVAL 60 DAY
  ) THEN 1 END) AS retained_60d,

  COUNT(CASE WHEN (
    SELECT SUM(settlement_amount) FROM tb_p_c_settlement s
    WHERE s.cpa_account = c.account
      AND s.bonus_time BETWEEN c.register_time AND c.register_time + INTERVAL 30 DAY
  ) > 0 THEN 1 END) AS revenue_30d,

  COUNT(CASE WHEN EXISTS (
    SELECT 1 FROM tb_p_cpa_account_relation r
    WHERE r.parent_cpa_account = c.account
      AND r.bind_time BETWEEN c.register_time AND c.register_time + INTERVAL 30 DAY
  ) THEN 1 END) AS referred
FROM cohort c;
```

---

## 模型 3 · 客户转化漏斗

### 漏斗阶段

```
注册 (REGISTER)
   ↓
开账号 (LIVE)
   ↓
合格 (QUALIFY)
   ↓
重复入金 / 重复交易
```

### 漏斗 SQL（按月）

```sql
SELECT
  DATE_FORMAT(register_time, '%Y-%m') AS cohort_month,
  COUNT(*)                              AS registered,
  COUNT(live_time)                      AS lived,
  COUNT(qualify_time)                   AS qualified,
  ROUND(COUNT(live_time)    * 100.0 / COUNT(*),                  1) AS live_rate,
  ROUND(COUNT(qualify_time) * 100.0 / NULLIF(COUNT(live_time),0), 1) AS qualify_rate,
  ROUND(COUNT(qualify_time) * 100.0 / COUNT(*),                  1) AS overall_rate
FROM tb_p_client
WHERE register_time >= '2026-01-01'
  AND brand = 'AU'
GROUP BY cohort_month
ORDER BY cohort_month;
```

### 漏斗洞察示例

| 月份 | 注册 | 开账号 | 合格 | 开账号率 | 合格率 | 总转化 |
|---|---|---|---|---|---|---|
| 2026-01 | 1000 | 650 | 380 | 65% | 58% | 38% |
| 2026-02 | 1200 | 800 | 470 | 67% | 59% | 39% |
| 2026-03 | 1500 | 900 | 510 | 60% | 57% | 34% ⚠️ |

**洞察**：3 月开账号率下降 7pp → 调查是否落地页 / KYC 流程变化。

---

## 模型 4 · Cohort 留存分析

### 概念

把"同月注册的代理"作为一个 cohort，跟踪他们在第 0、1、2、3 月的留存率。

### Cohort 表 SQL

```sql
WITH cohorts AS (
  SELECT account,
         DATE_FORMAT(register_time, '%Y-%m') AS cohort_month
  FROM tb_p_cpa_account
  WHERE register_time >= '2026-01-01'
    AND brand = 'AU'
),
activity AS (
  SELECT cpa_account,
         DATE_FORMAT(bonus_date, '%Y-%m') AS active_month
  FROM tb_p_bonus_share_day
  GROUP BY cpa_account, active_month
)
SELECT
  c.cohort_month,
  COUNT(DISTINCT c.account) AS total,
  COUNT(DISTINCT CASE WHEN PERIOD_DIFF(REPLACE(a.active_month,'-',''), REPLACE(c.cohort_month,'-',''))=0 THEN c.account END) AS m0,
  COUNT(DISTINCT CASE WHEN PERIOD_DIFF(REPLACE(a.active_month,'-',''), REPLACE(c.cohort_month,'-',''))=1 THEN c.account END) AS m1,
  COUNT(DISTINCT CASE WHEN PERIOD_DIFF(REPLACE(a.active_month,'-',''), REPLACE(c.cohort_month,'-',''))=2 THEN c.account END) AS m2,
  COUNT(DISTINCT CASE WHEN PERIOD_DIFF(REPLACE(a.active_month,'-',''), REPLACE(c.cohort_month,'-',''))=3 THEN c.account END) AS m3
FROM cohorts c
LEFT JOIN activity a ON c.account = a.cpa_account
GROUP BY c.cohort_month;
```

### 输出表（直观）

| Cohort | 总数 | M0 | M1 | M2 | M3 |
|---|---|---|---|---|---|
| 2026-01 | 100 | 100% | 78% | 62% | 51% |
| 2026-02 | 120 | 100% | 80% | 65% | - |
| 2026-03 | 150 | 100% | 75% | - | - |
| 2026-04 | 140 | 100% | - | - | - |

读法：2026-01 注册的代理，到第 3 个月还有 51% 活跃。

### 留存优秀的 cohort 是什么样

- 注册 → 30 天内活跃比例 ≥ 80%
- 90 天留存 ≥ 50%
- 长期留存（180 天）≥ 30%

低于这些数字要警惕。

---

## 模型 5 · LTV 终身价值

### 简单公式

```
LTV = ARPU × 留存月数
```

- ARPU：单代理月均佣金
- 留存月数：1 / 月度流失率

### LTV SQL

```sql
WITH agent_revenue AS (
  SELECT cpa_account,
         AVG(monthly_bonus) AS arpu
  FROM (
    SELECT cpa_account,
           DATE_FORMAT(bonus_date,'%Y-%m') AS month,
           SUM(bonus_amount) AS monthly_bonus
    FROM tb_p_bonus_share_day
    GROUP BY cpa_account, month
  ) monthly
  GROUP BY cpa_account
),
churn AS (
  SELECT
    COUNT(*) AS total_agents,
    COUNT(CASE WHEN DATEDIFF(CURDATE(), last_active) > 30 THEN 1 END) AS churned
  FROM (
    SELECT cpa_account, MAX(bonus_date) AS last_active
    FROM tb_p_bonus_share_day
    GROUP BY cpa_account
  ) act
)
SELECT
  AVG(arpu) AS arpu,
  (SELECT churned * 1.0 / total_agents FROM churn) AS churn_rate,
  AVG(arpu) / (SELECT churned * 1.0 / total_agents FROM churn) AS ltv
FROM agent_revenue;
```

### LTV 用法

- 与 CAC（获客成本）对比：LTV / CAC > 3 才是健康
- 分层比较：不同来源、不同地区代理的 LTV 差异
- 指导投放：高 LTV 来源加投，低 LTV 来源砍掉

---

## 流失分析专项

### 流失定义

| 客户 | 60 天无入金、无交易 |
| 代理 | 60 天无佣金生成 |

### 流失诊断 SQL

```sql
-- 上月流失代理特征
SELECT
  account_type,
  COUNT(*) AS churned_count,
  AVG(DATEDIFF(last_bonus, register_time)) AS avg_lifecycle_days,
  AVG(total_bonus) AS avg_lifetime_value
FROM (
  SELECT
    a.account,
    a.account_type,
    a.register_time,
    MAX(s.bonus_time) AS last_bonus,
    SUM(s.settlement_amount) AS total_bonus
  FROM tb_p_cpa_account a
  LEFT JOIN tb_p_c_settlement s ON a.account = s.cpa_account
  WHERE a.brand = 'AU'
  GROUP BY a.account
  HAVING last_bonus < NOW() - INTERVAL 60 DAY
     AND last_bonus >= NOW() - INTERVAL 90 DAY  -- 上月流失
) churned
GROUP BY account_type;
```

### 流失访谈建议

- 抽取 10-20 个流失代理
- 用 lark-cli/lark-im 发邀请
- 5 个问题：为什么停？竞品在用什么？产品哪里不好？什么场景会回来？给团队的建议？

---

## 输出范式

复杂分析请输出：

```markdown
# 【用户分析】xxx 群体 yyyy-Q-q

## 1. 分析对象与范围
- 群体：xxx
- 时间窗：xxx
- 数据来源：xxx

## 2. 关键发现 TOP 3
1. 发现 X：数字 + 解释
2. 发现 Y：数字 + 解释
3. 发现 Z：数字 + 解释

## 3. 详细数据
（表格 + 图表）

## 4. 根因（如适用）
5 Whys 或 决策树

## 5. 行动建议
| 建议 | 负责人 | 截止 |
|---|---|---|

## 6. 附录
- SQL
- 调研访谈记录
```

---

## 与其他 Skill 协作

```
[运营提需求]
    ↓
[data-query]      拉真实数据
[本 Skill]        套模型分析
[campaign-review] 输出复盘
[lark-cli]        同步飞书 + 通知相关人
```

---

## 反例：常见错误

| 错误 | 改正 |
|---|---|
| 只看平均值 | 看分布（P50/P90/P99） |
| 忽视 cohort 差异 | 必须按月 cohort 分析 |
| 用绝对值不用百分比 | 转化率、留存率都要用百分比 |
| 没有对照组 | 改动效果分析必须有 A/B |
| 数据来源不交叉验证 | 关键数字至少 2 个来源对比 |

---

## 引用

- 关联 Skill：data-query, campaign-review, hytech-cpa, lark-cli
- 经典书目：《Lean Analytics》、《精益数据分析》
