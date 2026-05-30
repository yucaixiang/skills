---
name: data-query
description: 数据自助查询 Skill — 让运营 / 产品 / BD 不会 SQL 也能自助拉数。基于 hytech-cpa 真实数据模型，按业务问题自动生成 SQL，支持飞书表格读写。覆盖代理业绩、佣金、客户、出金等常见查询场景。当用户说"帮我查 / 拉一下 / 看一下数据"时调用。
version: 1.0.0
origin: SkillForge
---

# data-query · 数据自助查询

> 让"不会写 SQL"不再是查数据的门槛。运营 / 产品自然语言描述需求 → AI 生成 SQL → 一键执行 → 结果同步飞书。

---

## When to Activate

- 用户说"帮我查 / 拉一下 / 看一下 xx 数据"
- 用户说"统计 xx 代理 / xx 客户 / xx 时间段"
- 用户说"导出 xx 报表"
- 与飞书表格数据相关（自动联动 `lark-cli/lark-sheets`）

---

## 工作流：5 步生成查询

```
Step 1 · 理解业务问题       用户原话 → 业务字段映射
   ↓
Step 2 · 定位数据表          调 hytech-cpa Skill 找表
   ↓
Step 3 · 生成 SQL            按真实表结构和约束
   ↓
Step 4 · 自查 SQL 风险        全表扫描？敏感字段？性能？
   ↓
Step 5 · 执行 + 结果同步     发飞书 / 写入飞书表格
```

---

## hytech-cpa 常用查询库

### 查询 1 · 代理列表 + 业绩

```sql
SELECT
  a.account                AS 代理账号,
  u.email                  AS 邮箱,
  a.account_type           AS 类型,
  a.account_status         AS 状态,
  a.register_time          AS 注册时间,
  COUNT(DISTINCT acr.client_user_uid) AS 名下客户数,
  COUNT(DISTINCT CASE WHEN c.client_status = 'QUALIFY' THEN c.user_id END)
                           AS 合格客户数
FROM tb_p_cpa_account a
JOIN tb_p_cpa_user u ON a.user_id = u.user_id
LEFT JOIN tb_p_account_client_relation acr
       ON a.account = acr.parent_cpa_account AND acr.valid = 1
LEFT JOIN tb_p_client c ON acr.client_user_uid = c.user_id
WHERE a.brand = 'AU'
  AND a.account_status = 'VALID'
GROUP BY a.account
ORDER BY 合格客户数 DESC;
```

### 查询 2 · 某代理本月佣金明细

```sql
SELECT
  s.settlement_no          AS 结算单号,
  s.settlement_type        AS 类型,
  s.settlement_amount      AS 金额,
  s.currency               AS 币种,
  s.bonus_time             AS 产生时间,
  c.email                  AS 客户邮箱
FROM tb_p_c_settlement s
LEFT JOIN tb_p_client cl ON s.client_user_id = cl.user_id
LEFT JOIN tb_p_cpa_user c ON cl.user_id = c.user_id
WHERE s.cpa_account = ?
  AND s.bonus_time >= '2026-06-01'
  AND s.bonus_time <  '2026-07-01'
ORDER BY s.bonus_time DESC;
```

### 查询 3 · 月度出金统计（按状态）

```sql
SELECT
  DATE_FORMAT(create_time, '%Y-%m') AS 月份,
  apply_status                       AS 状态,
  COUNT(*)                           AS 笔数,
  SUM(pay_amount)                    AS 总金额,
  AVG(pay_amount)                    AS 平均金额
FROM tb_p_c_apply_payment
WHERE brand = 'AU'
  AND create_time >= '2026-01-01'
GROUP BY 月份, 状态
ORDER BY 月份 DESC, 笔数 DESC;
```

### 查询 4 · 异常出金（大额 / 高频）

```sql
-- 大额（> $100,000）
SELECT cpa_account, pay_amount, create_time, apply_status
FROM tb_p_c_apply_payment
WHERE pay_amount > 100000
  AND create_time > NOW() - INTERVAL 7 DAY
ORDER BY create_time DESC;

-- 高频（24h 内 > 5 次）
SELECT cpa_account, COUNT(*) AS cnt, SUM(pay_amount) AS total
FROM tb_p_c_apply_payment
WHERE create_time > NOW() - INTERVAL 1 DAY
GROUP BY cpa_account
HAVING cnt > 5
ORDER BY cnt DESC;
```

### 查询 5 · 分润明细（某代理收到的分润）

```sql
SELECT
  bs.master_account        AS 主账户,
  bs.child_account         AS 实际产生佣金的下级,
  bs.bonus_amount          AS 原始佣金,
  bs.share_amount          AS 分润金额,
  bs.bonus_date            AS 日期
FROM tb_p_bonus_share bs
WHERE bs.parent_account = ?
  AND bs.bonus_date >= '2026-06-01'
ORDER BY bs.bonus_date DESC;
```

### 查询 6 · 客户状态分布

```sql
SELECT
  client_status,
  COUNT(*)                 AS 客户数,
  COUNT(*) * 100.0 / (SELECT COUNT(*) FROM tb_p_client WHERE brand = 'AU') AS 占比
FROM tb_p_client
WHERE brand = 'AU'
GROUP BY client_status;
```

### 查询 7 · 转化漏斗（注册 → LIVE → QUALIFY）

```sql
SELECT
  DATE_FORMAT(register_time, '%Y-%m') AS 月份,
  COUNT(*) AS 注册数,
  COUNT(live_time) AS 开账号数,
  COUNT(qualify_time) AS 合格数,
  COUNT(live_time) * 100.0 / COUNT(*) AS 开账号率,
  COUNT(qualify_time) * 100.0 / NULLIF(COUNT(live_time), 0) AS 合格率
FROM tb_p_client
WHERE brand = 'AU'
  AND register_time >= '2026-01-01'
GROUP BY 月份
ORDER BY 月份;
```

### 查询 8 · 日度业绩报告

```sql
SELECT
  bonus_date,
  cpa_account,
  client_count,
  deposit_amount,
  bonus_amount,
  share_amount
FROM tb_p_bonus_share_day
WHERE bonus_date BETWEEN '2026-06-01' AND '2026-06-30'
  AND cpa_account = ?
ORDER BY bonus_date;
```

### 查询 9 · 协议签署状态

```sql
SELECT
  COUNT(*)                            AS 协议总数,
  SUM(agreement_status = 'S')         AS 已签数,
  SUM(agreement_status = 'I')         AS 未签数,
  SUM(agreement_status = 'S') * 100.0 / COUNT(*) AS 签署率
FROM tb_p_agreement
WHERE brand = 'AU';
```

### 查询 10 · 余额异常对账

```sql
-- 流水累计与余额表对比
WITH balance_calc AS (
  SELECT account,
         SUM(change_amount) AS calculated_balance
  FROM tb_p_c_account_balance_record
  WHERE brand = 'AU'
  GROUP BY account
)
SELECT
  ab.account,
  ab.available_balance + ab.locked_balance AS db_balance,
  bc.calculated_balance,
  (ab.available_balance + ab.locked_balance) - bc.calculated_balance AS diff
FROM tb_p_c_account_balance ab
JOIN balance_calc bc ON ab.account = bc.account
WHERE ABS((ab.available_balance + ab.locked_balance) - bc.calculated_balance) > 0.01;
```

---

## SQL 安全自查

### 必查项

| 风险 | 检查方法 |
|---|---|
| 全表扫描 | EXPLAIN 看是否走索引 |
| `SELECT *` | 必须改成明确字段 |
| 大表 UPDATE/DELETE 无 LIMIT | 必须加 LIMIT |
| 大表无 WHERE | 必须加范围限制 |
| `LIKE '%xxx'` | 索引失效 |
| 跨品牌混查 | 必须 WHERE brand = ? |
| 跨数据源 JOIN | MySQL 不支持，必须改用 Feign 取数后 join |

### SQL 上线 Checklist

```
□ 走索引（EXPLAIN 看 type ≠ ALL）
□ 加 WHERE brand 隔离品牌
□ 时间范围限定（避免拉全量）
□ 输出字段明确（不用 *）
□ 涉及敏感字段（手机/邮箱）已脱敏或权限校验
□ 性能预估：单次查询数据量 < 10w 行
□ 高频查询需走 read-service 而非主库
```

---

## 与飞书联动

### 场景 1 · 查完直接发飞书

```bash
# 查询数据
result=$(mysql -e "SELECT ...")

# 发飞书
lark-cli im send --chat "#运营组" \
  --content "本月代理业绩 TOP10：$result"
```

### 场景 2 · 写入飞书表格

```bash
# 查询数据 → CSV
mysql -e "..." > result.csv

# 上传到飞书表格
lark-cli sheets append --sheet "https://x.larksuite.com/sheets/xxx" \
  --file result.csv --range "Sheet1!A1"
```

### 场景 3 · 自动生成日报表格

```bash
# 每天 9 点
lark-cli sheets update --sheet "..." \
  --range "Sheet1!A2:E100" --data "$(mysql ...)"
```

---

## 常见自然语言到 SQL 的映射

| 用户原话 | 翻译 |
|---|---|
| "看看张三这个月赚了多少" | 按 cpa_account 查 settlement，统计 settlement_amount |
| "前 10 名代理是哪些" | 按 settlement_amount 排序 LIMIT 10 |
| "异常出金有哪些" | 查询 4 |
| "本月新增多少代理" | tb_p_cpa_account where register_time |
| "客户都来自哪些国家" | tb_p_client GROUP BY country |
| "对账有差异的代理" | 查询 10 |
| "ABC 代理的下级有谁" | tb_p_cpa_account_path WHERE path LIKE '%/ABC/%' |
| "本月每天的佣金趋势" | tb_p_bonus_share_day GROUP BY date |

---

## 数据库连接信息

| 环境 | 连接 | 权限 |
|---|---|---|
| dev | mysql://localhost:3306 | 全部 |
| test | mysql://test.cpa.internal:3306 | 全部 |
| prod-readonly | mysql://prod-read.cpa.internal:3306 | 只读 ⭐ |
| prod-write | mysql://prod.cpa.internal:3306 | 需审批 ⚠️ |

> **运营 / 产品 / 测试 默认只能用 prod-readonly**。涉及生产写操作必须走审批流程。

---

## 反例

| 误用 | 后果 | 改正 |
|---|---|---|
| `SELECT * FROM tb_p_client` | 拉全表 OOM | 加 WHERE + LIMIT |
| `WHERE name LIKE '%张%'` | 全表扫描 | 用精确匹配或 ES |
| 跨品牌混查 | 数据泄露 | 加 brand 过滤 |
| 在主库跑大查询 | 影响业务 | 走 read-service |
| 把查询结果发给业务方时含手机号明文 | 信息泄露 | 脱敏 |

---

## 与其他 Skill 协作

```
[用户提需求]
    ↓
[本 Skill] 生成 SQL + 安全自检
    ↓
[执行] DB 查询
    ↓
[lark-cli] 结果同步飞书（消息/表格/文档）
    ↓
[campaign-review] / [user-analysis] 进一步分析
```

---

## 引用

- 关联 Skill：hytech-cpa, lark-cli, campaign-review, user-analysis
- 数据模型：hytech-cpa Skill 的 `docs/03-data-model.md`
