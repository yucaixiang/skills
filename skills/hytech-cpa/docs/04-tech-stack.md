# 04 · 技术栈与规范

> 本文档汇总 hytech-cpa 用到的技术组件、版本、关键配置以及命名/使用规范。
> 配置 Redis Key、Kafka Topic、Feign Client、告警之前必读。

---

## 一、技术栈总览

| 类别 | 组件 | 版本/说明 |
|---|---|---|
| 语言 | Java | 17+ |
| 框架 | Spring Boot | 3.x |
| 微服务 | Spring Cloud | 2023.x |
| 服务注册发现 | Consul | 配合 `consul.tags` 多品牌区分 |
| 服务调用 | OpenFeign | 通过 `*-platform-api` / `*-internal-api` |
| 持久化 | MyBatis | XML Mapper |
| 数据库 | MySQL | 主业务库 |
| 多数据源 | dynamic-datasource | MT4/MT5/业务库切换 |
| 缓存/锁 | Redis + Redisson | 分布式锁 + 缓存 |
| 规则引擎 | QlExpress | 动态规则执行 |
| 对象映射 | MapStruct | Entity ↔ DTO |
| 消息队列 | Kafka | 事件下游通知 |
| 定时任务 | Xxl-Job | 分布式调度 |
| 分布式 ID | 美团 Leaf | `hytech-cpa-leaf` 模块 |
| API 文档 | Knife4j | Swagger 增强 |
| 告警 | 自研 hytech-cpa-alert | 飞书卡片 |
| 导出 | EasyExcel | 大数据量导出 |
| JSON | Jackson + Fastjson | 优先 Jackson |
| 工具 | Hutool | 部分场景 |
| Lombok | Lombok | 全项目使用 |

---

## 二、Redis 使用规范

### 2.1 Key 命名格式

**格式**：`域:业务:模块:功能:数据`

- 全小写
- 冒号 `:` 分隔
- 描述性 + 简洁

### 2.2 命名示例

| 用途 | Key 格式 |
|---|---|
| 线索信息缓存 | `partner:cpa:leads:info:{leadsId}` |
| 线索分配锁 | `partner:cpa:leads:allocate:lock:{leadsId}` |
| 佣金计算锁 | `partner:cpa:bonus:calculate:lock:{clientUserId}` |
| 客户摘要缓存 | `partner:cpa:client:summary:{clientUserId}` |
| 规则缓存 | `partner:cpa:rule:detail:{ruleId}` |
| 操作任务键 | `OPERATION_TASK_KEY::{taskId}`（业务现有命名） |
| 强制 QFTD 标记 | `FORCE_QFTD_KEY::{clientUserId}`（业务现有命名） |

### 2.3 Redisson 分布式锁标准写法

```java
@Resource
private RedissonLock redissonLock;

public void calculateBonus(Long clientUserId) {
    String lockKey = String.format("BONUS_LOCK::%s", clientUserId);
    try {
        boolean locked = redissonLock.lock(lockKey, 30, TimeUnit.SECONDS);
        if (!locked) {
            log.warn("get lock failed, clientUserId:{}", clientUserId);
            throw new BizException(JobResponseEnum.LOCK_FAILED);
        }
        // 业务逻辑
    } finally {
        redissonLock.unlock(lockKey);
    }
}
```

### 2.4 缓存使用建议

- 高频读 / 低频写：缓存 5~30 分钟
- 列表数据：单条缓存，不缓存整个列表
- 缓存失效：双写策略（更新 DB 同步删 Cache）
- 缓存击穿防护：用 Redisson 锁配合

---

## 三、Kafka 使用规范

### 3.1 Topic 命名格式

**格式**：`域.业务.模块.功能`

- 全小写
- 点号 `.` 分隔

### 3.2 Topic 示例

| 用途 | Topic |
|---|---|
| 线索分配 | `partner.cpa.leads.allocate` |
| 佣金计算 | `partner.cpa.bonus.calculate` |
| QFTD 触发 | `partner.cpa.bonus.qftd` |
| 用户同步 | `partner.cpa.user.sync` |
| 关系变更 | `partner.cpa.relation.change` |

Topic 常量定义在 `TopicConstant` 类中，**严禁裸字符串**。

### 3.3 生产者标准写法

```java
@Resource(name = "kafkaTemplate")
private KafkaTemplate<String, String> kafkaTemplate;

public void sendBonusEvent(BonusMessage message) {
    String json = JsonUtil.toJsonString(message);
    kafkaTemplate.send(TopicConstant.BONUS_TOPIC, message.getKey(), json)
        .whenComplete((result, ex) -> {
            if (ex != null) {
                log.error("send kafka failed, topic:{}, message:{}",
                          TopicConstant.BONUS_TOPIC, json, ex);
                larkCommMessagePusher.sendAlert(LarkAlertRequest.builder()
                    .title("Kafka 发送失败")
                    .description("topic: " + TopicConstant.BONUS_TOPIC)
                    .build());
            }
        });
}
```

### 3.4 消费者标准写法

```java
@KafkaListener(topics = TopicConstant.BONUS_TOPIC,
               groupId = "${spring.application.name}")
public void onBonusMessage(ConsumerRecord<String, String> record) {
    BonusMessage msg = JsonUtil.parseObject(record.value(), BonusMessage.class);
    MDC.put("eventid", msg.getEventId());
    try {
        // 业务处理
    } catch (Exception e) {
        log.error("consume bonus message failed", e);
        // 重试 / 落到死信队列
    } finally {
        MDC.clear();
    }
}
```

---

## 四、告警模块 hytech-cpa-alert

### 4.1 模块定位

通用告警/通知模块，目前以飞书（Lark）卡片为主，可扩展企业微信、钉钉。

**核心抽象**：`CommMessagePusher` 接口
- `sendAlert(LarkAlertRequest)` — 严重错误（红卡片）
- `sendWarning(LarkAlertRequest)` — 警告（橙卡片）
- `sendInfo(LarkAlertRequest)` — 通知（蓝卡片）

### 4.2 引入方式

`pom.xml` 添加：

```xml
<dependency>
    <groupId>com.hytech</groupId>
    <artifactId>hytech-cpa-alert</artifactId>
</dependency>
```

引入后通过 Spring Boot 自动装配（`MessagePusherAutoConfiguration`）+ Consul 配置生效，**零业务侵入**。

### 4.3 配置（在 Consul 配置中心）

```yaml
cpa:
  alert:
    type: lark                # 当前仅支持 lark
  lark:
    appId: cli_xxx
    appSecret: xxxx
    chatId: oc_xxx            # 默认通知群
    bitable:
      appToken: xxx           # 多维表格 token
      tableId: tblxxx         # 责任人配置表
      cacheTtl: 300           # 责任人缓存秒数
```

### 4.4 调用示例

```java
@Resource
private CommMessagePusher larkCommMessagePusher;

public void onBonusCalculationError(ClientSummaryDto clientSummary) {
    larkCommMessagePusher.sendAlert(LarkAlertRequest.builder()
        .title("佣金计算")                              // ⚡ 用于匹配责任人
        .description(LarkTitleConstant.RULE_ERROR
                   + " ：ftd type qualified multiple bonus items.")
        .requestParams("clientSummary: " + JsonUtil.toJsonString(clientSummary))
        .build());
}
```

### 4.5 责任人自动 @

`hytech-cpa-alert` 通过 **多维表格（Bitable）** 配置责任人：
- 按 `LarkAlertRequest.title` 字段匹配多维表格中的 `title` 行
- 读取该行的 `people` 字段 → 卡片中自动 @
- 读取该行的 `MsgTag` 字段 → 显示 "重要问题 / 警告消息 / 通知消息"
- 责任人配置变更**无需发布代码**

### 4.6 卡片字段规范

所有告警走统一卡片（v2 + 1:2 列布局）：

| 左列字段 | 右列内容 |
|---|---|
| Service | `spring.application.name`（自动） |
| Event Id | MDC `eventid`（自动） |
| Title | 业务标题 |
| Error/Warn/Msg | description |
| MsgTag | 来自多维表格的标签 |
| Owner | 来自多维表格的责任人 |
| Request Params | 折叠面板 |
| Error Log | 折叠面板 |

### 4.7 常用 title 常量

定义在 `LarkTitleConstant`：

```
RULE_ERROR              规则错误
FTD_AMOUNT_REFRESH      FTD 金额刷新
BONUS_CALCULATION_FAIL  佣金计算失败
SETTLEMENT_FAIL         结算失败
PAYMENT_FAIL            支付失败
USER_SYNC_FAIL          用户同步失败
APPROVE_TIMEOUT         审批超时
KAFKA_SEND_FAIL         Kafka 发送失败
```

---

## 五、Feign 调用规范

### 5.1 服务接口定义

放在 `*-api` 模块：

```java
// internal-api（同域内调用）
@FeignClient(name = "${app.cpa-job.backend.service-name}")
public interface JobInternalApi {

    @PostMapping(value = "/bck/operation-task/create",
                 contentType = MediaType.APPLICATION_JSON_VALUE)
    InvokeResult<Long> createTask(@RequestBody OperationTaskDto dto);
}

// platform-api（跨域调用）
@FeignClient(name = "${app.cpa-job.backend.service-name}")
public interface PlatformJobExternalApi {

    @PostMapping("/bck/user/query")
    InvokeResult<List<DapUserInfoResp>> queryUserInfoList(
            @RequestBody List<Integer> userIds);
}
```

### 5.2 调用 InvokeResult 标准

`com.hytech.common.result.InvokeResult` 是公司标准返回体：

```java
InvokeResult<T>:
    private Integer code;
    private String message;
    private T data;

    public boolean success() { ... }
    public T getOrThrow() { ... }
```

调用示例：

```java
InvokeResult<List<DapUserInfoResp>> result =
    platformJobExternalApi.queryUserInfoList(userIds);

if (!result.success()) {
    log.error("queryUserInfoList failed: {}", result.getMessage());
    throw new BizException(JobResponseEnum.EXTERNAL_API_FAIL);
}

List<DapUserInfoResp> users = result.getData();
```

### 5.3 超时与重试

默认配置（可通过 `application.yml` 覆盖）：

```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 10000
spring:
  cloud:
    openfeign:
      retryer:
        period: 100
        maxPeriod: 1000
        maxAttempts: 3
```

---

## 六、Xxl-Job 定时任务规范

### 6.1 任务类命名

- 后缀 `Job`，如 `BatchJob`、`AccountBalanceJob`
- 位于 `cpa-job-executor/src/main/java/com/hytech/cpa/job/executor/{module}/`

### 6.2 标准模板

```java
@Slf4j
@Component
public class BonusBatchJob extends AbstractBatchJob {

    @XxlJob("bonusBatchJob")
    public void execute() throws Exception {
        String param = XxlJobHelper.getJobParam();
        log.info("BonusBatchJob start, param: {}", param);

        try {
            List<EventDto> events = loadEvents();
            eventHandleCoreService.handle(events);
            XxlJobHelper.handleSuccess("processed " + events.size() + " events");
        } catch (Exception e) {
            log.error("BonusBatchJob failed", e);
            larkCommMessagePusher.sendAlert(LarkAlertRequest.builder()
                .title("定时任务失败")
                .description("BonusBatchJob: " + e.getMessage())
                .build());
            XxlJobHelper.handleFail(e.getMessage());
        }
    }
}
```

### 6.3 任务命名规范

| 模块 | 命名前缀 |
|---|---|
| 佣金计算 | `bonus*Job` |
| 结算/支付 | `payment*Job`、`balance*Job` |
| 用户同步 | `user*Job` |
| 客户转移 | `transfer*Job` |
| 报表 | `report*Job` |
| 校验/补偿 | `verify*Job`、`compensation*Job` |

---

## 七、规则引擎 QlExpress

### 7.1 用途

动态执行规则表达式，主要用在：
- QFTD 条件判断（"入金 >= X 且 交易量 >= Y"）
- 额外奖励规则判断（"ROI >= X 且 客户数 ∈ [min, max]"）

### 7.2 标准用法

```java
@Resource
private ExpressRunner expressRunner;

DefaultContext<String, Object> context = new DefaultContext<>();
context.put("ftdAmount", ftdAmount);
context.put("liveDays", liveDays);
context.put("tradeVolume", tradeVolume);

Object result = expressRunner.execute(expression, context, null, true, false);
boolean qualified = Boolean.TRUE.equals(result);
```

### 7.3 表达式示例

```
ftdAmount >= 100 && liveDays <= 90 && tradeVolume >= 1.0
```

存储位置：`RuleQftdConditionEntity.expression`

---

## 八、MapStruct 转换规范

### 8.1 Convert 接口位置

`com.hytech.cpa.core.backend.model.convert.{业务包}`

### 8.2 标准模板

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface ClientSummaryConvert {
    ClientSummaryConvert INSTANCE = Mappers.getMapper(ClientSummaryConvert.class);

    ClientSummaryDto entityToDto(ClientSummaryEntity entity);
    ClientSummaryEntity dtoToEntity(ClientSummaryDto dto);

    List<ClientSummaryDto> entitiesToDtos(List<ClientSummaryEntity> entities);
}
```

### 8.3 字段不一致时的映射

```java
@Mapping(source = "cpaAccount", target = "accountId")
@Mapping(source = "bonusTime", target = "createTime")
SettlementDto entityToDto(SettlementEntity entity);
```

---

## 九、Knife4j 接口文档

### 9.1 访问地址

- Admin: `http://localhost:9101/doc.html`
- Client: `http://localhost:9103/doc.html`
- Open: `http://localhost:9106/doc.html`

### 9.2 注解规范

```java
@Tag(name = "代理账户管理")
@RestController
@RequestMapping("/bsn/cpa-account")
public class BsnCpaAccountController {

    @Operation(summary = "代理账户分页查询")
    @PostMapping("/page")
    public InvokeResult<PageResult<CpaAccountRes>> page(
            @RequestBody @Valid CpaAccountPageReq req) {
        ...
    }
}
```

---

## 十、多品牌 / 多环境

### 10.1 启动参数

```
-Ddeploy.env=dev          部署环境
-Denv=dev                 业务环境
-Dbrand=AU                品牌
-Dconsul.tags=AU          Consul tag
-Dconsul.host=127.0.0.1
-Dconsul.port=8500
-Dconsul.service.tags=AU
-Dconsul.select.tag=AU    服务选择 tag
```

### 10.2 品牌隔离实现

- 服务发现：Consul `tag` 区分
- 配置中心：Consul KV 按品牌分目录
- 数据库：通常按品牌分库或分表
- Entity `brand` 字段：所有业务表都有 `brand` 列

### 10.3 代码中获取当前品牌

```java
@Value("${brand}")
private String brand;

// 或从 ThreadLocal / RequestContextHolder 取
String currentBrand = BrandContextHolder.getCurrent();
```

---

## 十一、日志规范

### 11.1 SLF4J + Logback

```java
@Slf4j
public class XxxService {
    public void doSomething(Long clientUserId) {
        log.info("doSomething start, clientUserId:{}", clientUserId);
        // ...
        log.info("doSomething end, clientUserId:{}, result:{}", clientUserId, result);
    }
}
```

### 11.2 MDC 链路追踪

```java
MDC.put("eventid", UUID.randomUUID().toString());
try {
    // 业务逻辑，所有日志自动带 eventid
} finally {
    MDC.clear();
}
```

Logback 配置中通过 `%X{eventid}` 输出。

### 11.3 关键字段

- `clientUserId` · 客户标识
- `cpaAccount` · 代理账户
- `eventid` · 事件链路
- `traceid` · 全链路（如有 SkyWalking 等接入）

### 11.4 日志级别使用

- `ERROR` — 影响业务的错误（必须告警）
- `WARN` — 异常分支但可恢复（如锁失败重试）
- `INFO` — 业务关键节点（生产打开）
- `DEBUG` — 详细排查信息（生产关闭）

---

## 十二、参考资料

- `数据迁移doc/hytech-cpa项目理解.md`
- `docs/hytech-cpa-alert使用说明.md`
- 多维表格查询功能使用说明.md
- 飞书告警自动@负责人完整实施方案.md
