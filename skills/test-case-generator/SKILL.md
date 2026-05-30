---
name: test-case-generator
description: 测试用例自动设计与生成 Skill — 基于需求文档 + 真实代码，按"等价类 / 边界值 / 状态机 / 流程覆盖"方法论生成结构化测试用例。支持功能用例、接口用例、异常用例、性能用例的统一模板。当测试 / 研发 / 产品需要"为 xx 需求 / xx 接口 / xx Service 设计测试用例"时调用。
version: 1.0.0
origin: SkillForge
---

# test-case-generator · 测试用例设计生成器

> 让 AI 同时读懂"需求 + 代码"，生成业务正确性与技术覆盖度兼顾的测试用例。

---

## When to Activate

- 用户说"帮我为 xx 需求设计测试用例"
- 用户说"为 xx 接口 / Service 生成用例"
- 用户说"覆盖一下 xx 业务的边界场景"
- 测试同学在 Claude Code 里输入需求文档 / Jira 链接
- 与 `api-test-automation` Skill 串联（先设计，再脚本化）

---

## 工作流：5 步生成测试用例

```
Step 1 · 读懂需求           PRD / Jira / 飞书文档
   ↓
Step 2 · 读懂代码           调用 code-walkthrough + 业务 Skill（如 hytech-cpa）
   ↓
Step 3 · 识别测试维度       功能 / 边界 / 异常 / 状态机 / 性能 / 兼容
   ↓
Step 4 · 应用设计方法论     等价类 / 边界值 / 决策表 / 场景法 / 状态迁移
   ↓
Step 5 · 输出结构化用例     标准模板 + 优先级 + 预期结果
```

---

## 标准用例模板

```markdown
### 用例 ID：CASE-{业务}-{序号}

| 字段 | 内容 |
|---|---|
| **用例名称** | 一句话描述 |
| **所属业务** | hytech-cpa / 佣金计算 / FTD 模式 |
| **关联需求** | PROJ-1234（PRD 第 3.2 节） |
| **关联代码** | `FTDBonusCalculator.calculate()` |
| **测试类型** | 功能 / 接口 / 异常 / 性能 / 状态机 |
| **优先级** | P0 主流程 / P1 重要 / P2 一般 / P3 增强 |
| **前置条件** | 1. 代理已签协议<br>2. 客户已开账号<br>3. 规则 R001 配置 FTD 模式 |
| **测试步骤** | 1. 客户入金 $200<br>2. 等候 30s 触发 BatchJob<br>3. 查询 tb_p_bonus_qftd |
| **测试数据** | clientUserId=10001, ftdAmount=200 |
| **预期结果** | 1. tb_p_bonus_qftd 出现 1 条记录<br>2. bonusAmount = $20<br>3. 不发飞书告警 |
| **回归测试** | 是 / 否 |
| **自动化覆盖** | 待自动化 / 已自动化（脚本 ID：AUTO-xxx） |
```

---

## 6 大测试设计方法

### 方法 1 · 等价类划分

把输入域分组，**每组选一个代表值**测试。

**例**：FTD 金额规则区间 `[100, 500)`、`[500, 1000)`、`[1000, ∞)`

| 等价类 | 代表值 | 用途 |
|---|---|---|
| 不合格（< 100） | $50 | 不应触发佣金 |
| 合格档位 1 | $200 | bonusAmount = $20 |
| 合格档位 2 | $700 | bonusAmount = $50 |
| 合格档位 3 | $5000 | bonusAmount = $500 |

### 方法 2 · 边界值

任何区间的**边界 ± 1** 是 bug 重灾区。

**例**：区间 `[100, 500)`

| 边界点 | 用例 | 预期 |
|---|---|---|
| $99 | 不合格 | 无佣金 |
| $100 | 刚好合格（左闭） | 命中档位 1 |
| $499 | 档位 1 上界（右开） | 命中档位 1 |
| $500 | 进入档位 2（右开变左闭） | 命中档位 2 |
| 极值 | $0.01 / $999999999 | 不崩溃，正确处理 |

### 方法 3 · 决策表

多条件组合时用。

**例**：是否触发佣金 = f(已签协议？规则有效？金额合格？多次QFTD开关？)

| # | 已签协议 | 规则有效 | 金额合格 | 多次QFTD | 预期 |
|---|---|---|---|---|---|
| 1 | ❌ | ✅ | ✅ | ✅ | 不触发（未签） |
| 2 | ✅ | ❌ | ✅ | ✅ | 不触发（规则过期） |
| 3 | ✅ | ✅ | ❌ | ✅ | 不触发（金额不合格） |
| 4 | ✅ | ✅ | ✅ | ❌ | 触发首次，第二次不触发 |
| 5 | ✅ | ✅ | ✅ | ✅ | 每次入金都触发 |

### 方法 4 · 状态迁移

含状态字段的实体，要测**每种合法/非法状态变化**。

**例**：`ClientEntity.clientStatus` 状态机

```
        合法迁移                       非法迁移
REGISTER ──首次入金──> LIVE             LIVE → REGISTER（不允许）
LIVE ──满足QFTD──> QUALIFY               QUALIFY → REGISTER（不允许）
                                         REGISTER → QUALIFY（跳过LIVE）
```

每条**合法迁移**写正例，每条**非法迁移**写反例。

### 方法 5 · 流程覆盖（场景法）

端到端业务流程，按"主路径 + 备选路径 + 异常路径"出用例。

**例**：出金审批流程

| 路径类型 | 场景 |
|---|---|
| 主路径 | 申请 → 审批通过 → 支付成功 → 余额减少 |
| 备选路径 1 | 申请 → 审批拒绝 → 锁定余额释放 |
| 备选路径 2 | 申请 → 审批超时 → 锁定余额释放 |
| 备选路径 3 | 申请后撤回 → 锁定余额释放 |
| 异常路径 1 | 申请时可用余额不足 → 拒绝创建 |
| 异常路径 2 | 审批通过但支付网关失败 → 申请回滚 |
| 异常路径 3 | 申请单号重复 → 幂等返回 |
| 异常路径 4 | 并发 2 次提交 → 仅 1 笔成功 |

### 方法 6 · 错误推测（基于经验）

凭测试经验直觉补充：

- 输入为 null
- 输入超长（10000 字符）
- 输入特殊字符（emoji、零宽字符、SQL 注入）
- 并发场景
- 网络异常 / DB 重启
- 时区 / 日期跨界（凌晨 0 点、跨月、跨年）
- 数值精度（BigDecimal 0.1 + 0.2）
- 集合空、单元素、超大

---

## 按代码类型生成用例

### 类型 A · Controller 接口用例

读完 Controller 后，列出：
- HTTP 方法 + URL
- 入参类型 + 字段
- 出参类型 + 字段
- 鉴权要求

**最少要测**：
- ✅ 正常入参，返回成功
- ✅ 必填字段缺失 → `@Valid` 校验失败
- ✅ 字段值非法（类型 / 长度 / 范围）→ 拒绝
- ✅ 无 token / token 过期 → 401/403
- ✅ 限流场景 → 429

### 类型 B · Service 业务用例

读完 Service 后，按业务规则出用例：

- ✅ 主流程正确
- ✅ 各分支正确（if-else / switch / 策略路由）
- ✅ 异常分支（抛 BizException）
- ✅ 并发场景（加锁是否生效）
- ✅ 事务一致性（中途失败是否回滚）

**例**：`BonusCalculationCoreServiceImpl.calculateQftdBonus`

```
✅ FTD 合格 → 写入 BonusQftd + Settlement
✅ Country 合格 → 同上
✅ Progressive 合格 → 同上
✅ rule.isDeactivate → 直接 return，不计算
✅ 非多次QFTD + 已计算过 → return
✅ FTD 命中多档 → 发飞书告警，不写记录
✅ ruleQftdBonusItems 为 null → 记 error log，不抛异常
✅ Redis 锁失败 → 抛 BizException
```

### 类型 C · Mapper / SQL 用例

读完 SQL 后：

- ✅ 数据存在 → 正确返回
- ✅ 数据不存在 → 返回 null / 空列表
- ✅ 软删数据 → 不应返回（如 deleted=1）
- ✅ 多品牌隔离 → brand=AU 查不到 brand=UK 的
- ✅ 索引命中 → 看执行计划
- ✅ 大数据量 → 走分页 / LIMIT
- ✅ NULL 字段 → 是否当成"未匹配"

### 类型 D · 异步任务 / Job 用例

- ✅ 正常执行成功
- ✅ 重复触发是否幂等
- ✅ 任务超时是否中断
- ✅ 中途失败是否补偿
- ✅ XxlJob 重试机制是否生效
- ✅ 失败发飞书告警

### 类型 E · Kafka 消费用例

- ✅ 消息格式正确 → 正常处理
- ✅ 消息格式损坏 → 死信队列
- ✅ 业务处理失败 → 重试
- ✅ 重复消息 → 幂等
- ✅ 消费速度跟不上 → 监控告警

---

## 与代码 + 需求双输入的具体做法

### 输入示例

**需求**：
> PROJ-5678 · 支持代理批量为客户重新绑定关系
>
> 后台运营录入一个 Excel（A 列代理账号，B 列客户 userId 列表），系统校验通过后，
> 批量解除旧绑定（valid=0），新建新绑定（valid=1）。

**代码**：
- `BsnTransferClientController.batchTransfer`
- `BsnTransferClientService.batchTransfer`
- `AccountClientRelationRepository.softDelete` / `insertBatch`

### AI 该生成的用例（节选）

```markdown
### CASE-TRANSFER-001 · 单条批量绑定成功
前置：代理 #1001 存在；客户 #100001 已绑定代理 #999（valid=1）
步骤：
  1. 上传 Excel：行 1 = (1001, [100001])
  2. 触发 batchTransfer
预期：
  1. tb_p_account_client_relation 中：
     - 旧 (parent=999, client=100001) valid 改为 0
     - 新 (parent=1001, client=100001) valid=1
  2. 接口返回 success_count=1
  3. 记 operation_record_detail
优先级：P0

### CASE-TRANSFER-002 · 多客户批量
前置：代理 #1001；客户 #100001, #100002, #100003 已分别绑定其他代理
步骤：
  1. 上传 Excel：行 1 = (1001, [100001, 100002, 100003])
  2. 触发 batchTransfer
预期：3 条新绑定生效，3 条旧绑定 valid=0
优先级：P0

### CASE-TRANSFER-003 · 代理不存在
步骤：上传 Excel：行 1 = (99999999, [100001])
预期：返回 fail_count=1，错误码 CPA_ACCOUNT_NOT_FOUND
优先级：P0

### CASE-TRANSFER-004 · 客户不存在
步骤：上传 Excel：行 1 = (1001, [99999999])
预期：fail_count=1，错误码 CLIENT_NOT_FOUND
优先级：P0

### CASE-TRANSFER-005 · 客户已绑定到目标代理（重复绑定）
前置：客户 #100001 已绑定代理 #1001（valid=1）
步骤：上传 Excel：行 1 = (1001, [100001])
预期：幂等成功，无新增记录
优先级：P1

### CASE-TRANSFER-006 · Excel 格式错误
步骤：上传非 .xlsx 文件
预期：返回 400，错误码 INVALID_FILE_FORMAT
优先级：P1

### CASE-TRANSFER-007 · Excel 超大（5000 行）
预期：分批处理，全部成功，<60s 完成
优先级：P1

### CASE-TRANSFER-008 · 并发提交相同 Excel
预期：第二次返回"任务进行中"或幂等
优先级：P2

### CASE-TRANSFER-009 · 中途失败回滚
模拟：处理第 50 条时 DB 抛异常
预期：前 49 条已生效不回滚（默认设计）/ 或全部回滚（取决于 PRD）
优先级：P0（必须明确）

### CASE-TRANSFER-010 · 跨品牌
前置：代理 #1001（brand=AU），客户 #100001（brand=UK）
预期：拒绝，错误码 BRAND_NOT_MATCH
优先级：P1
```

---

## 用例覆盖率自检

设计完用例后，按这个 checklist 自检：

```
✅ 主流程已覆盖
✅ 边界值已覆盖（每个区间的 ±1）
✅ 错误分支已覆盖（抛异常的每个 case）
✅ 并发场景已考虑
✅ 数据一致性已考虑（事务、流水、关联表）
✅ 跨业务影响已考虑
✅ 安全场景已考虑（鉴权、注入、越权）
✅ 性能边界已考虑（大数据量、超时）
✅ 兼容性已考虑（旧数据、多品牌、多语言）
✅ 回滚 / 重试 / 幂等已覆盖
```

---

## 输出格式选择

根据使用场景生成不同形式：

| 场景 | 输出格式 |
|---|---|
| 测试管理平台（如 TestRail / Xray） | Markdown 表格 |
| 飞书多维表格 | CSV / Excel（含字段映射） |
| 直接给开发 self-check | Checklist 列表 |
| 转成自动化脚本 | 用例 ID + 步骤 JSON → 交给 `api-test-automation` |

---

## 协作链路

```
[读懂业务] → hytech-cpa Skill / PRD
       ↓
[读懂代码] → code-walkthrough Skill
       ↓
[设计用例] → 本 Skill (test-case-generator)
       ↓
[评审用例] → 测试 Leader 评审 / AI 自检
       ↓
[落地脚本] → api-test-automation Skill
       ↓
[执行 + 回归] → CI / 测试平台
```

---

## 注意事项

- ❌ 不要"为了凑数"出无价值的用例（如：输入合法值多 5 个变体）
- ❌ 不要忽略"业务规则隐含约束"（如：金额必须 > 0，但 PRD 没写）
- ❌ 不要只测正向，反向用例同样重要
- ✅ 一个用例只验证一个点（断言不要堆 10 个）
- ✅ 每个用例必须可独立运行（不依赖前一个用例的副作用）
- ✅ 用例命名说人话（CASE-FTD-EDGE-100 vs "ftd 边界 100"）

---

## 引用

- 《软件测试的艺术》Glenford Myers
- 《Google 软件测试之道》James Whittaker
- 业务上下文：`hytech-cpa` Skill
- 自动化下游：`api-test-automation` Skill
