---
name: compliance-check
description: 合规自检清单 Skill — 在产品 / 研发 / 运营做新功能、新规则、新报送、新接口时，自助进行合规检查。涵盖数据隐私、AML / KYC、跨境数据、监管报送、操作审计 5 大维度。基于 hytech-cpa 真实业务约束。当涉及"合规 / 监管 / 隐私 / AML / KYC"等场景时调用。
version: 1.0.0
origin: SkillForge
---

# compliance-check · 合规自检清单

> 合规不是事后救火，而是事前体检。让产品 / 研发 / 运营在动手前就能自助识别红线。

---

## When to Activate

- 设计涉及"用户数据 / 资金 / 监管 / 跨境"的功能
- 产品评审、技术评审前的预审
- 监管报送变更
- 新的对外开放接口

---

## 5 大合规维度

```
1. 数据隐私       PII / GDPR / 隐私法案合规
2. AML / KYC      反洗钱、客户身份识别、可疑交易上报
3. 跨境数据       数据本地化、跨境传输路径
4. 监管报送       监管字段、报送频次、准确性
5. 操作审计       关键操作日志、双人复核、可追溯
```

---

## 维度 1 · 数据隐私

### 红线

| 场景 | 检查 |
|---|---|
| 存储 PII（姓名 / 邮箱 / 手机 / 身份证 / 银行卡） | 必须加密 + 访问审计 |
| 日志输出 PII | 必须脱敏（如 138****1234） |
| 接口出参含 PII | 按角色 / 权限脱敏 |
| 跨服务传输 PII | TLS + 权限校验 |
| PII 数据导出 | 必须有审计记录 + 老板审批 |

### 自查 Checklist

```
□ 涉及的字段是否包含 PII？
□ PII 字段在 DB 存储是否加密？
□ 日志中 PII 是否脱敏？
□ 接口出参的脱敏规则是什么？
□ 是否有访问审计日志？
□ 数据保留期限是多久？（GDPR 要求合理期限）
□ 用户能否申请删除自己的数据？
```

### 反例

| 反例 | 改正 |
|---|---|
| 日志直接 `log.info("user: " + user)`（含手机号） | 脱敏：`log.info("user: {}", user.maskedPhone())` |
| 接口出参原样返回 idCardNumber | 仅 KYC 角色可见，其他角色看到 `****` |
| Excel 导出含全字段 PII | 限制可下载角色 + 加密 zip |

---

## 维度 2 · AML / KYC

### 必查项

| 场景 | 检查 |
|---|---|
| 客户首次入金 | 必须完成 KYC（身份验证） |
| 大额交易 | 触发 AML 监测 + 报送 |
| 异常出金 | 触发风控冻结（参考 `risk-control`） |
| 高风险地区 | 加强尽调（EDD） |
| PEP（政治敏感人物） | 特别审查 |
| 可疑交易 | 上报金融情报机构（如 AUSTRAC for AU） |

### 自查 Checklist

```
□ 涉及的资金流是否会触发 AML 阈值？
□ KYC 状态是否检查？
□ 黑名单（OFAC / 联合国制裁名单）是否过滤？
□ 是否有可疑交易上报机制？
□ AML 报告是否定期归档？
```

### hytech-cpa 具体关联

- 代理 KYC：`tb_p_cpa_user` 的 KYC 状态字段
- 客户 KYC：通过 DAP/User 服务同步
- 异常监测：`risk-control` Skill 的 6 大场景
- 大额出金：`ApplyPaymentRiskService.preCheck`

---

## 维度 3 · 跨境数据

### 红线

| 场景 | 红线 |
|---|---|
| 中国境内用户数据 | 不出境（数据本地化） |
| 欧盟 GDPR 用户数据 | 跨境需 SCC / BCR |
| 美国用户数据 | 看州法（CCPA 等） |
| 澳洲 AU | Privacy Act 1988 |

### 自查 Checklist

```
□ 数据存储地点在哪？
□ 跨境传输路径是什么？
□ 是否有用户同意？
□ 是否有合规通道（如 AWS Bedrock）？
□ 是否走 SCC / BCR / 适当性认证？
```

### hytech-cpa 实践

- 多品牌（AU / 其他）数据按品牌分库 / 分 schema
- AI 调用走 AWS Bedrock（合规通道）
- 内部数据不直接对接外部第三方分析服务

---

## 维度 4 · 监管报送

### 常见报送场景

| 报送 | 频次 | 字段 |
|---|---|---|
| 月度业务报送 | 月 | 代理数、客户数、交易量、佣金总额 |
| 异常交易报送 | 实时 | 可疑交易 |
| 年度审计 | 年 | 全量数据快照 |
| 监管要求字段 | 按要求 | 监管机构 / 执照号 |

### 自查 Checklist

```
□ 涉及的字段是否影响监管报送？
□ 改动是否需要通知合规团队？
□ 报送格式是否需要更新？
□ 报送系统是否能正确读取新字段？
□ 历史数据是否需要回填？
```

### hytech-cpa 关联

- `tb_p_client.regular` · 所属监管机构
- `tb_p_client.license` · 监管执照
- `LicenseEnum` / `RegulatorEnum` · 枚举定义

---

## 维度 5 · 操作审计

### 必查项

| 场景 | 检查 |
|---|---|
| 涉及资金操作 | 双人复核 + 完整审计日志 |
| 涉及代理冻结 / 解冻 | 操作记录 + 原因 + 审批 |
| 涉及规则变更 | 变更前后快照 + 操作人 + 时间 |
| 涉及数据导出 | 谁、什么时候、导了什么 |

### 自查 Checklist

```
□ 涉及的关键操作是否有审计日志？
□ 日志是否包含：who / when / what / from where？
□ 日志保留时长？
□ 重要操作是否需要双人复核？
□ 涉及金额修改是否需要 BPM 审批？
□ 日志是否防篡改？
```

### hytech-cpa 实践

- `tb_p_operation_record` + `tb_p_operation_record_detail` · 操作日志
- `OperationRecordBusinessTypeEnum` · 业务类型枚举
- BPM 审批：所有重要操作走 `ApproveWorkEntity`

---

## 完整自检报告模板

```markdown
# 【合规自检】xxx 需求

## 1. 自检维度结果

| 维度 | 信号 | 备注 |
|---|---|---|
| 1. 数据隐私 | 🟢 / 🟡 / 🔴 | xxx |
| 2. AML/KYC  | 🟢 / 🟡 / 🔴 | xxx |
| 3. 跨境数据 | 🟢 / 🟡 / 🔴 | xxx |
| 4. 监管报送 | 🟢 / 🟡 / 🔴 | xxx |
| 5. 操作审计 | 🟢 / 🟡 / 🔴 | xxx |

## 2. 总体结论

✅ 合规 / ⚠️ 需调整 / ❌ 不合规

## 3. 关键风险

1. xxx
2. xxx

## 4. 改进建议

1. xxx
2. xxx

## 5. 需合规团队评审

- [ ] 是
- [ ] 否
```

---

## 与其他 Skill 协作

```
[产品 / 研发出方案]
       ↓
[本 Skill] 合规自检
       ↓ 🟢 通过 → 继续
       ↓ 🟡 调整 → 回到 prd-template
       ↓ 🔴 阻塞 → 升级到合规团队
       ↓
[需要看业务] → hytech-cpa Skill
[需要看代码] → code-walkthrough Skill
[需要通知]   → lark-cli Skill
```

---

## 引用

- 关联 Skill：hytech-cpa, risk-control, prd-template, lark-cli
- 监管法规：所在地区监管要求（AU 等）
- 隐私：GDPR / CCPA / Privacy Act 1988
- AML：FATF Recommendations
