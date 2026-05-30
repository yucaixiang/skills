# 01 · 整体架构

> 本文档说明 hytech-cpa 的整体架构、4 大业务领域划分、4 层分层规则、跨域通信机制和故障隔离策略。

---

## 一、设计哲学

hytech-cpa 采用 **业务域纵向切分 + 技术层横向分层** 的双维度架构。

```
┌─────────────────────────────────────────────────────────────────┐
│                       4 个独立业务领域                          │
├──────────────────┬──────────────────┬──────────────────┬────────┤
│      ToB         │       ToC        │      ToJ         │  ToO   │
│   Admin 后台     │   Client 自助    │   Job 异步       │  Open  │
│   (9101)         │   (9103)         │   (9105)         │ (9106) │
└──────┬───────────┴──────┬───────────┴──────┬───────────┴───┬────┘
       │                  │                  │               │
   ─── 每个领域内部都遵循下面 4 层分层 ───
       │                  │                  │               │
   ┌───▼──────────────────▼──────────────────▼───────────────▼────┐
   │  application 应用服务层    协议转换、技术适配                │
   │  business    业务逻辑层    服务编排、跨业务整合              │
   │  backend     数据基础层    本业务数据原子访问                │
   │  api         服务协议层    Feign + 数据契约                  │
   └──────────────────────────────────────────────────────────────┘
```

设计目标：
- **业务隔离**：4 个领域物理/逻辑隔离，**故障不跨域**
- **职责单一**：4 层职责清晰，**禁止层级穿透**
- **可独立演进**：每个领域可独立部署、独立扩缩容
- **多品牌支持**：通过启动参数 `-Dbrand=AU` 区分品牌

---

## 二、4 大业务领域

### 2.1 ToB · `hytech-cpa-admin`（端口 9101）

**用途**：公司运营、管理员使用的后台系统。

**典型功能**：
- 代理账户管理（账户列表、升级申请、关系维护、转移）
- 客户管理（客户列表、CRM 同步、转移、批量操作）
- 佣金规则配置（规则组、Tier、品种分组、额外奖励）
- 协议管理（默认协议、协议签署状态）
- 结算审批（结算单、出金审批）
- 报告查询（业绩报告、佣金报告、分润报告）

**子模块结构**：
```
hytech-cpa-admin/
├── cpa-admin-api                  # 接口契约（仅 model + 接口定义）
│   ├── cpa-admin-api-model        # API 数据契约
│   ├── cpa-admin-base-api         # API 公共接口
│   ├── cpa-admin-internal-api     # 业务域内（其他 ToB）调用入口
│   └── cpa-admin-platform-api     # 业务域外调用入口
└── cpa-admin-service              # 服务实现
    ├── cpa-admin-application      # 启动入口
    ├── cpa-admin-backend          # 数据基础服务（/bck/）
    └── cpa-admin-business         # 综合业务服务（/bsn/）
```

**启动参数示例**：
```
-Ddeploy.env=dev
-Denv=dev
-Dbrand=AU
-Dconsul.tags=AU
-Dconsul.host=127.0.0.1
-Dconsul.port=8500
-Dconsul.service.tags=AU
-Dconsul.select.tag=AU
```

### 2.2 ToC · `hytech-cpa-client`（端口 9103）

**用途**：代理 IB 登录后的自助门户。

**典型功能**：
- 代理个人信息查看
- 代理客户列表查询
- 佣金规则与协议自助签署
- 结算单查询与出金申请
- 分润查询、邀请码管理
- 业绩报告（自己名下）

**子模块结构**：与 admin 类似，多一个 **read-service** 用于读写分离：
```
cpa-client-service          # 主服务（含写）
cpa-client-read-service     # 只读服务（用于大查询）
```

只读服务启动多一个参数：
```
-Dspring.cloud.consul.discovery.auto-register=true
```

### 2.3 ToJ · `hytech-cpa-job`（端口 9105）

**用途**：定时任务执行 + MQ 消费 + 跨领域基础数据服务。

**子模块结构**：
```
hytech-cpa-job/
├── cpa-job-api
│   ├── cpa-job-api-model
│   ├── cpa-job-base-api
│   ├── cpa-job-internal-api
│   └── cpa-job-platform-api
├── cpa-job-backend-service     # 仅基础服务，无 business 层
└── cpa-job-executor            # XxlJob 任务执行器 + Kafka 消费者
```

**典型任务**（位于 `cpa-job-executor`）：
- `bonus/BatchJob` · 佣金批量计算
- `bonus/VerifyJob` · 佣金校验
- `bonus/CompensationJob` · 佣金补偿
- `payment/AccountBalanceJob` · 账户余额计算
- `user/UserSyncJob` · 与 DAP/User 同步
- `transfer/BulkTaskTransferJob` · 批量客户转移

> **特殊性**：Job 域是唯一**没有 business 层**的领域，仅提供 `backend-service`。

### 2.4 ToO · `hytech-cpa-open`（端口 9106）

**用途**：对外开放接口，提供给公司其他系统集成。

**典型场景**：
- 其他 toB 系统（如 CRM）查询代理-客户关系
- 第三方报表系统拉取佣金数据
- 财务系统对账

---

## 三、4 层分层规则（核心）

### 3.1 application · 应用服务层

**职责**：技术适配 + 启动

**允许做**：
- 集成 backend 和 business
- 实现技术关注点：认证、日志、监控、限流
- 提供 REST 端点
- Spring Boot 启动入口

**禁止做**：
- ❌ 包含业务规则
- ❌ 直接调用 Mapper

**典型 Bean**：`CpaAdminApplication`、`CpaClientApplication`

### 3.2 business · 业务逻辑层

**职责**：服务编排 + 跨业务数据整合

**允许做**：
- 集成其他领域的 platform-api（Feign 调用）
- 组合本业务 backend 的数据，实现"复合查询"
- 调用 BPM、Alert、Leaf 等横切模块

**禁止做**：
- ❌ 直接访问其他业务的数据库
- ❌ 向其他业务的 business 提供接口

**Controller URL 前缀**：`/bsn/`

### 3.3 backend · 数据基础层

**职责**：本业务数据原子访问

**允许做**：
- MyBatis CRUD 操作
- Redis 缓存读写（仅本业务数据）
- 定义 backend 内部使用的 CoreService

**禁止做**：
- ❌ **直接对外返回 DB 实体（Entity / PO）**，必须用 DTO
- ❌ 包含其他业务的接口或数据契约
- ❌ 在 backend 里集成外部 API

**Controller URL 前缀**：`/bck/`

### 3.4 api · 服务协议层

**职责**：接口契约 + 数据契约

**允许做**：
- 用 Feign 包装服务名和 backend URI
- 携带数据契约 Model

**禁止做**：
- ❌ 包含本业务数据验证规则（验证应在 business 层）
- ❌ 包含具体业务逻辑

**模块划分**：
- `*-api-model` · 数据契约（Req/Res/DTO）
- `*-base-api` · 公共接口
- `*-internal-api` · 域内调用入口
- `*-platform-api` · 跨域调用入口

---

## 四、跨域通信机制

### 4.1 域内通信（同领域）

```
[ToB business] ──Feign──> [ToB internal-api] ──HTTP──> [ToB backend]
```

- 通过 `cpa-{domain}-internal-api` 包定义 Feign 接口
- 注解 `@FeignClient(name = "cpa-{domain}-backend-service")`
- 走 Consul 服务发现

### 4.2 跨域通信（不同领域）

```
[ToC business] ──Feign──> [ToB platform-api] ──HTTP──> [ToB backend or business]
```

- 通过 `cpa-{otherDomain}-platform-api` 包调用
- 不直接调用 internal-api（internal 仅限本域）
- 走 Consul 服务发现

### 4.3 与外部系统通信

```
[业务 business] ──Feign──> [外部系统 Feign Client]
```

外部系统典型集成：
- **DAP/User 服务**：通过 `PlatformJobExternalApi.queryUserInfoList()` 等获取用户基础信息
- **MT4/MT5 数据库**：多数据源直接 SQL 查询交易和持仓数据
- **CRM**：通过 `PlatformAdminExternalApi.queryLeafSales()` 等获取销售关系
- **Cellxpert 历史 API**：迁移期使用，位于 `hytech-cpa-cellxpert-api-test`

---

## 五、故障隔离与稳定性

### 5.1 故障域隔离

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│  ToB 故障  │    │  ToC 故障  │    │  ToJ 故障  │    │  ToO 故障  │
└─────┬──────┘    └─────┬──────┘    └─────┬──────┘    └─────┬──────┘
      │ 不传染            │ 不传染          │ 不传染          │ 不传染
      ↓                  ↓                ↓                ↓
   只影响本域服务，不会跨域影响其他领域
```

实现方式：
- **网络层**：VPC 分段 + 安全组策略
- **架构层**：禁止跨域 service 调用，仅通过标准化 API + DTO 交互
- **数据库层**：每个领域独立数据库（或独立 Schema）

### 5.2 容错策略

| 场景 | 策略 |
|---|---|
| 外部 API 调用失败 | OpenFeign 默认重试 + Sentinel 熔断（视配置） |
| Kafka 消费失败 | 重试 + 死信队列 |
| 定时任务失败 | XxlJob 自动重试 + 告警（Lark） |
| Redis 锁超时 | 业务侧 catch + 降级 |

### 5.3 告警机制

所有关键失败都走 `hytech-cpa-alert` 模块发飞书卡片：
```java
larkCommMessagePusher.sendAlert(LarkAlertRequest.builder()
        .title("佣金计算")
        .description("规则错误：FTD 命中多条规则")
        .requestParams("clientSummary: " + JsonUtil.toJsonString(clientSummary))
        .build());
```

告警自动 @ 责任人，责任人配置在多维表格中，按 title 字段匹配。

---

## 六、多品牌支持

通过启动参数 `-Dbrand=AU` 实现：
- Consul 注册时携带 `tag=AU`
- 服务发现时按 `consul.select.tag` 过滤
- 数据库 / 配置中心按品牌隔离

各品牌共用同一份代码，差异通过配置中心管理。

---

## 七、参考资料

- `README.md` · 项目根目录的整体架构说明
- `数据迁移doc/hytech-cpa项目理解.md` · 完整业务理解文档
- `数据迁移doc/迁移设计流程图-汇报版.md` · 迁移设计图
- `docs/hytech-cpa-alert使用说明.md` · 告警模块详解
