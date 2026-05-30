---
name: oncall-trading
description: hytech-cpa 系统值班手册 — 针对代理佣金管理系统的 7×24 值班 SOP。涵盖告警分级、5 步通用排查法、7 个典型故障 SOP、应急操作工具箱、战时群规范、复盘流程。值班同学接到任何告警时调用，AI 引导从告警 → 定位 → 处置 → 复盘的全流程。
version: 2.0.0
origin: SkillForge
---

# oncall-trading · hytech-cpa 值班手册

> 让值班从"经验依赖"升级为"AI 引导的标准化作业"。半夜被电话叫醒的新值班也能稳住场子。

---

## When to Activate

- 用户说"接到 xx 告警"、"线上有问题"
- 用户描述故障现象（"佣金没算出来"、"代理出金失败"、"接口报错"）
- 飞书告警群被 @ 进来
- 凌晨被电话叫醒
- 需要按 SOP 查日志 / 看监控 / 操作恢复

---

## 值班角色与职责

| 角色 | 职责 | 联络方式 |
|---|---|---|
| 一线值班 | 接告警、初步排查、按 SOP 处置 | 飞书 @当班值班 |
| 业务 Owner | 业务复杂场景决策 | 飞书联系 |
| Tech Lead | 重大故障 / 架构问题决策 | 飞书 + 电话 |
| SRE / DBA | 基础设施故障 | 飞书 SRE 群 |
| 安全值班 | 安全事件 | 飞书安全群 |
| 合规值班 | 监管 / 资金安全 | 飞书合规群 |

> 责任人配置在多维表格中（参考 `hytech-cpa-alert` 模块）。告警自动 @ 对应人。

---

## 告警分级

### P0 · 重大故障（5 分钟响应）

- 全站不可用
- 资金 / 数据错误且影响代理实际收入
- 安全漏洞被利用
- 监管报送失败

**操作**：立即接手 → 拉飞书战时群 → @ Tech Lead + 业务 Owner → 启动故障应急（参考 `incident-response` Skill）

### P1 · 严重影响（15 分钟响应）

- 核心功能不可用（如：佣金计算停止）
- 单一品牌不可用
- 部分代理出金失败
- 重要接口大面积超时

**操作**：5 分钟内确认 → 15 分钟内有初步定位 → 1 小时内恢复

### P2 · 部分影响（30 分钟响应）

- 非核心功能异常
- 少量数据异常（< 10 条）
- 报表延迟
- 单个代理投诉

**操作**：30 分钟内查明 → 工作日内修复

### P3 · 体验问题（2 小时响应）

- 界面卡顿
- 文案错误
- 单一用户体验问题

**操作**：记录 → 排期修复

---

## 通用排查 SOP（5 步法）

```
Step 1 · 看告警卡片（30 秒）
  ↓ 提取关键信息：title、service、eventid、owner、错误日志
Step 2 · 看监控（1 分钟）
  ↓ Grafana 大盘：QPS、RT、错误率、JVM、DB
Step 3 · 查日志（2 分钟）
  ↓ 按 eventid 串联整条链路
Step 4 · 定位代码 / 数据 / 配置
  ↓ 用 code-walkthrough Skill 找代码源头
Step 5 · 处置 → 通报 → 复盘
```

### Step 1 · 看告警卡片

```
告警卡片关键字段（hytech-cpa-alert）：
  Service:    hytech-cpa-admin-service
  Event Id:   abc-123
  Title:      佣金计算
  MsgTag:     重要问题
  Owner:      @xiaodong.wang
  Error/Warn: ftd type is qualified multiple bonus items.
  Params:     { clientSummary: ... }
```

→ 从 title 知道是"佣金计算"模块
→ 从 owner 知道找谁
→ 从 eventid 串链路

### Step 2 · 看监控

| 大盘 | 看什么 |
|---|---|
| 服务概览 | QPS / RT / 错误率 / JVM |
| DB 监控 | 慢查询、连接池、主从延迟 |
| Redis 监控 | 命中率、内存、阻塞 |
| Kafka 监控 | lag、消息堆积 |
| 业务监控 | 佣金生成数、出金申请数 |

异常信号：
- QPS 突增/突降
- RT 飙升
- 错误率 > 1%
- DB 慢查询 > 阈值
- Kafka lag 持续上涨

### Step 3 · 查日志

```bash
# 按 eventid 查整条链路
grep "abc-123" /logs/hytech-cpa-*/*.log

# 按业务标识查
grep "clientUserId:100001" /logs/hytech-cpa-*.log

# 看 ERROR
grep "ERROR" /logs/hytech-cpa-*.log | tail -100
```

### Step 4 · 定位源头

用 `code-walkthrough` Skill 反向定位代码：

```bash
# 异常消息找定义
grep -rn "ftd type is qualified multiple bonus items" --include="*.java"
# → FTDBonusCalculator.java:71

# 看上下文理解为何触发
```

### Step 5 · 处置

| 处置类型 | 操作 |
|---|---|
| 数据问题 | 修数据脚本（必须先备份）+ 灰度验证 |
| 配置问题 | Consul 改配置 + 验证 + 通知 |
| 代码 bug | 紧急 hotfix 分支 + 评审简化 + 灰度 |
| 基础设施 | 通知 SRE / DBA |
| 外部依赖 | 切换备用 / 降级 |

---

## 典型故障 SOP 库

### 故障 1 · 佣金计算停止（无新佣金生成）

**排查 SOP**：
1. 看 cpa-job-executor 服务是否在线（Consul）
2. 看 XxlJob 控制台 `bonusBatchJob` 最后执行时间
3. 看 ERROR 日志 + 飞书告警群
4. 查输入：MT4/MT5 数据是否正常拉取？
   ```bash
   tail -100 /logs/cpa-job-executor/*.log | grep "BatchJob"
   ```
5. 查 ClientSummary 表是否在更新

**常见原因**：
- XxlJob 任务被禁用 → 启用
- MT4/MT5 数据库连接失败 → 看连接池配置
- Kafka 消费阻塞 → 看 lag
- Redis 锁泄漏 → 重启释放

### 故障 2 · 代理出金申请失败

**排查 SOP**：
1. 看代理具体错误消息
2. 查代理余额：
   ```sql
   SELECT * FROM tb_p_c_account_balance WHERE account = ?;
   ```
3. 看是否有进行中的出金（`tb_p_c_apply_payment` status=P）
4. 看 BPM 审批流程是否正常
5. 看支付网关返回

**常见原因**：
- 可用余额不足（含锁定余额） → 业务正常
- 同时多次提交 → RedissonLock 拦截，提示"操作中"
- BPM 流程未启动 → 看 ApproveProcessEntity 配置
- 支付网关故障 → 暂停审批 + 通知

### 故障 3 · FTD 命中多档告警

**排查 SOP**：
1. 看告警 description 中的 clientSummary
2. 查规则配置：
   ```sql
   SELECT * FROM tb_p_rule_qftd_bonus_item WHERE rule_id = ?;
   ```
3. 检查区间是否重叠（如 [100, 500) 和 [400, 1000)）

**处置**：
- 暂停该规则计算（先告警，不写错记录）
- 通知业务 Owner 修复规则配置
- 修复后人工触发受影响客户的补偿计算

### 故障 4 · 余额对不上

**排查 SOP**：
1. 查余额表当前值
2. 查流水累计
3. 对比：流水累计 == available + locked？
4. 不一致 → 找断点

```sql
-- 流水累计
SELECT SUM(change_amount)
FROM tb_p_c_account_balance_record
WHERE account = ?;

-- 当前余额
SELECT available_balance + locked_balance
FROM tb_p_c_account_balance
WHERE account = ?;
```

**处置**：
- 找出断点（哪笔写库失败）
- 补偿 + 通知财务 / 合规
- **绝对不允许直接改余额表，必须通过流水补偿**

### 故障 5 · 接口大面积超时

**排查 SOP**：
1. 看 Grafana 接口监控
2. 看 DB 慢查询
3. 看 Redis 命中率
4. 看 Feign 下游服务

**处置**：
- DB 慢查询 → 紧急加索引 / 限流
- Redis 击穿 → 启用降级
- 下游故障 → 熔断 + 降级
- 流量异常 → 限流 + 检查是否被刷

### 故障 6 · Kafka 消费堆积

**排查 SOP**：
1. 看消费组 lag
2. 看消费者实例是否在线
3. 看消费者错误日志

**处置**：
- 消费者掉线 → 重启
- 单条消息处理慢 → 优化 / 扩容
- 全部失败 → 看是否中毒消息（先跳过）

### 故障 7 · 用户同步异常

**排查 SOP**：
1. 看 `UserSyncJob` 执行日志
2. 看与 DAP 服务的 Feign 调用是否成功
3. 看具体失败的用户

**处置**：
- DAP 接口超时 → 通知用户中心团队
- 部分用户失败 → 跳过 + 补偿任务
- 全部失败 → 暂停同步 + 告警

---

## 应急操作工具箱

### 工具 1 · 紧急关闭告警

```bash
# 通过多维表格调整责任人为"暂停告警"标签
# 或通过 Consul 改 cpa.alert.type 为 none
```

### 工具 2 · 紧急流量切换

```bash
# 通过 Consul 改服务 tag，把流量切到只读
consul kv put cpa/admin/readonly true
```

### 工具 3 · 紧急回滚

```bash
# K8s 回滚
kubectl rollout undo deployment/cpa-admin-service

# 数据回滚（极少用）
# 必须备份 → 业务确认 → 灰度
```

### 工具 4 · 紧急扩容

```bash
kubectl scale deployment/cpa-job-executor --replicas=10
```

### 工具 5 · 紧急关闭某个 Job

```bash
# 在 XxlJob 控制台禁用某个任务
# 或修改任务 cron 为远期
```

---

## 战时群规范

P0/P1 故障时通过 `lark-cli` 自动建战时群：

```bash
lark-cli im create-chat \
  --name "#P0-佣金计算停止-0531" \
  --members "@xiaodong.wang @luchen.li @sre-oncall"
```

群规：
- ✅ 只发与故障相关的信息
- ✅ 每 15 分钟同步进展
- ✅ 决策必须有明确的负责人确认
- ❌ 不允许闲聊
- ❌ 不允许并发讨论无关问题

---

## 复盘流程

故障恢复 24 小时内：

```markdown
# 故障复盘：[故障标题]

## 1. 时间线
| 时间 | 事件 |
|---|---|
| 02:00 | 告警触发 |
| 02:02 | 值班接手 |
| 02:15 | 定位原因 |
| 02:35 | 恢复 |

## 2. 影响范围
- 时长：35 分钟
- 影响代理：xxx 人
- 影响金额：$xxx
- 用户感知：xxx

## 3. 根因分析
（五个为什么）

## 4. 处置过程
按步骤记录做了什么

## 5. 改进项
| 改进项 | 负责人 | 截止时间 |
|---|---|---|
| xxx | xxx | xxx |

## 6. 预防同类问题
xxx
```

复盘文档用 lark-cli 同步到飞书知识库。

---

## 值班心法

### 心法 1 · 慢就是快

- 别慌，看清楚再动手
- 错误操作比不操作更糟
- 不确定就升级（找 Tech Lead）

### 心法 2 · 数据是宝藏

- 改数据前必须备份
- 改完留证据（截图 + SQL）
- 涉及资金必须双人复核

### 心法 3 · 沟通透明

- 进度每 15 分钟同步
- 不藏问题
- 业务方需要知道"什么时候能恢复"

### 心法 4 · 复盘必做

- 复盘不是甩锅
- 复盘是为了别人不再踩同样的坑
- 改进项必须落地（不写抽象的"加强意识"）

---

## 与其他 Skill 协作

```
[接告警] → 本 Skill 引导
       ↓
[需要看代码] → code-walkthrough Skill
       ↓
[需要查业务] → hytech-cpa Skill
       ↓
[严重故障升级] → incident-response Skill
       ↓
[操作飞书]   → lark-cli Skill（建战时群、同步进度、写复盘）
```

---

## 引用

- 关联 Skill：hytech-cpa, code-walkthrough, incident-response, lark-cli
- 告警链路：hytech-cpa-alert 模块说明
- 监控大盘：Grafana / Prometheus
