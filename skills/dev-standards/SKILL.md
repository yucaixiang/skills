---
name: dev-standards
description: 公司 Java 后端通用开发规范 — 融合《阿里巴巴 Java 开发手册》经典条款 + hytech-cpa 项目真实落地实践 + 业界通用最佳实践。涵盖命名、代码风格、注释、日志、异常、并发、集合、SQL、安全、单测、Git/PR 流程。在写代码、改代码、重构、生成代码模板时必须遵守。
version: 2.0.0
origin: SkillForge
---

# 公司 Java 后端开发规范

> 本规范是公司 Java 后端开发的**红线集合**：违反 P0 红线会导致 PR 驳回，违反 P1 黄线需在 PR 中说明理由。
> 灵感来源：《阿里巴巴 Java 开发手册（黄山版）》核心条款 + hytech-cpa 项目真实落地经验 + 业界共识。

---

## When to Activate

- 编写、修改、重构、Review 任何 Java 后端代码
- 创建新模块、新类、新接口
- 配置 Maven 依赖、Spring Bean、数据库连接
- 写 SQL、Redis Key、Kafka Topic
- 任何"我应该这样写吗？"的疑问

---

## 红线分级

| 级别 | 含义 | 违反后果 |
|---|---|---|
| 🔴 **P0** | 红线，绝对禁止 | PR 直接驳回 |
| 🟡 **P1** | 黄线，原则禁止 | 需在 PR 描述中说明理由 |
| 🟢 **P2** | 建议，鼓励遵守 | Review 时讨论 |

---

## 一、命名规范

### 🔴 P0 · 类名 / 方法名 / 变量名严禁使用拼音、缩写、单字母

```java
// ❌ 禁止
class JsbService { }                  // 拼音
class CpaUsrSvc { }                   // 过度缩写
String s = "...";                     // 单字母
int n = 100;                          // 无意义

// ✅ 正确
class CpaUserService { }
String userName = "...";
int retryCount = 100;
```

### 🔴 P0 · 强制大小写约定

| 类型 | 规范 | 示例 |
|---|---|---|
| 类名 | UpperCamelCase | `CpaAccountService` |
| 方法名 / 变量 / 参数 | lowerCamelCase | `calculateBonus` `userId` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| 包名 | 全小写，不用下划线 | `com.hytech.cpa.bonus` |
| 抽象类 | `Abstract` 前缀 | `AbstractBatchJob` |
| 异常类 | `Exception` 后缀 | `BizException` |
| 测试类 | `Test` 后缀 | `BonusCalculatorTest` |
| 枚举 | `Enum` 后缀 | `CpaAccountTypeEnum` |
| 接口与实现 | 实现加 `Impl` 后缀 | `XxxService` + `XxxServiceImpl` |

### 🟡 P1 · 业务概念使用全名

- ❌ `addr`、`mng`、`stmt` → ✅ `address`、`manage`、`statement`
- 例外：领域内公认的缩写可用（如 `id`、`url`、`json`、`xml`、`uuid`、`okr`）

### 🟢 P2 · Boolean 变量不加 `is` 前缀

> 阿里规范：POJO 类中的 boolean 字段，**不要**加 `is` 前缀，否则部分框架反序列化会出错。

```java
// ❌ 不推荐
private Boolean isDeleted;

// ✅ 推荐
private Boolean deleted;
```

---

## 二、包结构与分层（强制）

```
com.{company}.{project}.{domain}.{service-type}.
├── config/         配置类
├── constant/       静态常量
├── controller/     控制层
├── enums/          枚举
├── jobs/           定时任务
├── mapper/         MyBatis Mapper
├── model/
│   ├── req/        Req 入参
│   ├── res/        Res 出参
│   ├── db/         Entity（DB 映射）
│   └── dto/        DTO（层间传输）
├── service/        Service 接口
│   └── impl/       Service 实现
└── util/           工具类
```

### 🔴 P0 · Controller 严禁包含业务逻辑

Controller 只做：参数校验、调用 Service、组装返回。**禁止**在 Controller 里写循环、判断、SQL。

### 🔴 P0 · Service 严禁直接返回 Entity 给外部调用方

跨层（如 backend → business、business → controller）必须用 DTO/Res 隔离。

### 🟡 P1 · Service 单类不超过 20 个 public 方法

超过 20 个说明职责过重，应拆分。

---

## 三、代码风格

### 🔴 P0 · 单方法不超过 80 行（建议 50 行）

超过必须拆。否则 Review 时驳回。

### 🔴 P0 · 单类不超过 800 行（建议 500 行）

超过说明类承担太多职责，必须拆分。

### 🟡 P1 · 缩进 4 空格，禁止 Tab

IDE 设置统一为 4 空格。

### 🟡 P1 · 行长度不超过 120 字符

超过强制换行。

### 🟢 P2 · 文件结尾保留空行

避免某些工具报警。

---

## 四、注释规范

### 🔴 P0 · 类、接口必须有 Javadoc

```java
/**
 * 佣金计算核心服务
 *
 * @author xiaodong.wang
 * @date 2025-08-26
 */
public interface BonusCalculationCoreService { }
```

### 🟡 P1 · 复杂方法必须有方法 Javadoc

```java
/**
 * 计算 QFTD 佣金。
 * 
 * @param clientSummary 客户摘要（必须已加载规则）
 * @throws BizException RULE_NOT_EXISTS 规则不存在
 * @throws BizException LOCK_FAILED Redis 锁获取失败
 */
public void calculateQftdBonus(ClientSummaryDto clientSummary) { }
```

### 🔴 P0 · 严禁注释失效

代码改了，注释没改 → 误导维护者。**改代码 = 检查相邻注释**。

### 🟢 P2 · TODO/FIXME 必须带负责人和日期

```java
// TODO(xiaodong.wang 2025-09-01): 支持多币种汇率转换
// FIXME(luchen.li 2025-08-20): 这里在高并发下会有竞态
```

---

## 五、依赖注入

### 🔴 P0 · 优先构造注入

```java
// ✅ 推荐
@Service
@RequiredArgsConstructor
public class CpaAccountServiceImpl {
    private final CpaAccountRepository cpaAccountRepository;
    private final RedissonLock redissonLock;
}
```

### 🟡 P1 · 可接受 `@Resource`

```java
// 历史代码常见，可保留
@Resource
private CpaAccountRepository cpaAccountRepository;
```

### 🟢 P2 · 新代码不再用 `@Autowired`

构造注入更易测、支持 final。

---

## 六、Lombok 使用

### 推荐组合

| 角色 | 注解 |
|---|---|
| Entity（DB） | `@Getter @Setter @SuperBuilder @AllArgsConstructor @NoArgsConstructor @ToString(callSuper = true) @EqualsAndHashCode(callSuper = true)` |
| DTO / Req / Res | `@Data @Builder @AllArgsConstructor @NoArgsConstructor` |
| Service / Controller | `@Slf4j @RequiredArgsConstructor` |

### 🟡 P1 · 禁止在 Entity 上用 `@Data`

`@Data` 包含 `@EqualsAndHashCode`，继承场景下会导致父类字段不参与比较。Entity 用单独的 `@Getter @Setter`。

### 🔴 P0 · 禁止用 `@SneakyThrows` 吞 IOException 等受检异常

```java
// ❌ 危险
@SneakyThrows
public String readFile() { return Files.readString(path); }

// ✅ 正确：受检异常显式处理
public String readFile() throws IOException { ... }
```

---

## 七、集合 / 字符串

### 🔴 P0 · 集合判空用 `CollectionUtils.isEmpty()` / `isNotEmpty()`

```java
// ❌ 禁止
if (list != null && list.size() > 0) { }

// ✅ 正确
if (CollectionUtils.isNotEmpty(list)) { }
```

### 🔴 P0 · 字符串拼接用 StringBuilder（循环内）

```java
// ❌ 禁止（循环内每次创建 String）
String result = "";
for (String s : list) {
    result += s;
}

// ✅ 正确
StringBuilder sb = new StringBuilder();
for (String s : list) {
    sb.append(s);
}
```

### 🔴 P0 · `equals` 反向调用避免 NPE

```java
// ❌ 危险
if (status.equals("ACTIVE")) { }

// ✅ 正确
if ("ACTIVE".equals(status)) { }
// 或用 Objects.equals(status, "ACTIVE")
```

### 🟡 P1 · 包装类比较用 `equals`

```java
Long a = 1000L;
Long b = 1000L;
a == b  // ❌ false（超出缓存范围）
a.equals(b)  // ✅ true
```

---

## 八、异常处理

### 🔴 P0 · 抛业务异常用 `BizException` + ResponseEnum

```java
// ❌ 禁止
throw new RuntimeException("规则不存在");

// ✅ 正确
throw new BizException(JobResponseEnum.RULE_NOT_EXISTS);
```

ResponseEnum 按领域划分：`AdminResponseEnum` / `ClientResponseEnum` / `JobResponseEnum` / `OpenResponseEnum` / `BackendResponseEnum`。

### 🔴 P0 · 严禁裸 catch 吞异常

```java
// ❌ 禁止
try {
    doSomething();
} catch (Exception e) {
    // 什么都没做
}

// ✅ 正确
try {
    doSomething();
} catch (BizException e) {
    log.warn("doSomething biz fail: {}", e.getMessage());
    throw e;  // 必要时重抛
} catch (Exception e) {
    log.error("doSomething unexpected fail", e);
    larkCommMessagePusher.sendAlert(...);
    throw new BizException(JobResponseEnum.UNEXPECTED_ERROR);
}
```

### 🟡 P1 · 不在 Controller 层 catch 业务异常

抛出来，让 `GlobalExceptionHandler` 统一处理 → 标准 `InvokeResult` 返回。

### 🔴 P0 · finally 不放业务逻辑，只放资源释放

```java
// ✅ 正确
try {
    redissonLock.lock(key);
    // 业务
} finally {
    redissonLock.unlock(key);  // 资源释放
}
```

---

## 九、日志规范

### 🔴 P0 · 用 `@Slf4j` + 占位符

```java
// ❌ 禁止
log.info("user: " + user + ", time: " + time);  // 字符串拼接

// ✅ 正确
log.info("user:{}, time:{}", user, time);
```

### 🔴 P0 · 异常日志必须带堆栈

```java
// ❌ 错误
log.error("calc fail: " + e.getMessage());

// ✅ 正确
log.error("calc fail, clientUserId:{}", clientUserId, e);
```

### 🟡 P1 · 关键字段强制带业务标识

`clientUserId` / `cpaAccount` / `eventid` 等。

### 🟢 P2 · 日志级别

| 级别 | 用途 | 生产开关 |
|---|---|---|
| ERROR | 影响业务的错误（必告警） | ON |
| WARN | 异常分支但可恢复 | ON |
| INFO | 业务关键节点 | ON |
| DEBUG | 详细排查 | OFF |
| TRACE | 极细粒度（很少用） | OFF |

### 🔴 P0 · 严禁打印敏感信息

- 密码、Token、Secret、银行卡号 → 必须脱敏
- 手机号、邮箱 → 部分掩码（`138****1234`）
- 身份证号、客户全名 → 视监管要求处理

---

## 十、并发

### 🔴 P0 · 共享变量必须线程安全

```java
// ❌ 危险
private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");  // 非线程安全

// ✅ 正确
private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd");
```

### 🔴 P0 · 分布式锁用 Redisson，单机锁用 ReentrantLock

```java
// 分布式（推荐）
redissonLock.lock(key, 30, TimeUnit.SECONDS);

// 单机
private final ReentrantLock localLock = new ReentrantLock();
```

### 🟡 P1 · 线程池必须命名 + 设置拒绝策略

```java
// ❌ 禁止
Executors.newFixedThreadPool(10);  // 默认线程名 + 无界队列

// ✅ 正确
ThreadFactory factory = new ThreadFactoryBuilder()
    .setNameFormat("bonus-calc-%d").build();
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    10, 50, 60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(1000),
    factory,
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

### 🔴 P0 · `@Transactional` 不可自调用

Spring AOP 限制：同类内 A 方法调 B 方法（B 有 `@Transactional`）不会触发事务。需把 B 抽到独立 Bean。

参考 hytech-cpa 的 `BonusTransactional`：独立 Bean 包装事务。

---

## 十一、数据库 / SQL

### 🔴 P0 · 必须使用预编译，禁止字符串拼 SQL

```xml
<!-- ❌ 禁止 -->
<select id="findByName">
    SELECT * FROM user WHERE name = '${name}'
</select>

<!-- ✅ 正确 -->
<select id="findByName">
    SELECT * FROM user WHERE name = #{name}
</select>
```

### 🔴 P0 · 严禁 `SELECT *`

明确字段名，便于变更追溯、节省网络。

### 🟡 P1 · 大表 UPDATE/DELETE 必须有 WHERE 且加 LIMIT

```sql
-- ❌ 危险
DELETE FROM tb_p_event_payment WHERE status = 'DONE';

-- ✅ 安全
DELETE FROM tb_p_event_payment WHERE status = 'DONE' LIMIT 1000;
```

### 🔴 P0 · 索引字段不要在表达式里

```sql
-- ❌ 索引失效
WHERE DATE(create_time) = '2026-01-01'

-- ✅ 走索引
WHERE create_time >= '2026-01-01' AND create_time < '2026-01-02'
```

### 🟡 P1 · 分页查询必须有 ORDER BY

否则跨页可能重复 / 漏数据。

### 🟢 P2 · 批量插入用一条 SQL

```java
// ❌ 慢
for (Entity e : list) mapper.insert(e);

// ✅ 快
mapper.insertBatch(list);
```

---

## 十二、Redis / Kafka

### 🔴 P0 · Key/Topic 用常量，禁止裸字符串

```java
// ❌ 禁止
redisTemplate.opsForValue().set("user:" + id, value);

// ✅ 正确
redisTemplate.opsForValue().set(String.format(RedisKeyConstant.USER_INFO, id), value);
```

### 🔴 P0 · Redis Key 必须有 TTL（除特殊场景）

避免内存泄漏。

### 🔴 P0 · Kafka 消息必须有重试和死信处理

不能因为某条消息处理失败导致整个 topic 卡住。

详细命名规范见 [hytech-cpa Skill 的 04-tech-stack.md]。

---

## 十三、安全编码

### 🔴 P0 · 严禁硬编码密钥、密码

```java
// ❌ 禁止
private static final String DB_PWD = "123456";

// ✅ 正确：从配置中心 / 环境变量读取
@Value("${db.password}")
private String dbPwd;
```

### 🔴 P0 · 严禁明文存储敏感信息

密码 → BCrypt / SHA-256+盐
身份证 / 银行卡 → 加密存储 + 字段级权限

### 🔴 P0 · 接口必须有鉴权 + 限流

`@PreAuthorize` / Spring Security / 自定义注解。

### 🟡 P1 · XSS 防护（前端 + 后端双重）

后端：对用户输入做转义。
前端：渲染前 sanitize。

### 🟡 P1 · 文件上传校验

- MIME 类型白名单
- 文件大小限制
- 文件名重命名（避免路径穿越）

---

## 十四、单元测试

### 🟡 P1 · 核心业务逻辑覆盖率 ≥ 70%

特别是 Calculator、Service、规则引擎、状态机。

### 🔴 P0 · 测试方法命名 `should_xxx_when_xxx`

```java
@Test
void should_throw_bizException_when_rule_not_found() { }

@Test
void should_return_zero_bonus_when_ftd_amount_below_threshold() { }
```

### 🔴 P0 · 测试必须可重复运行

- 不依赖外部网络
- 不依赖某个时刻（用 Mock 时钟）
- 不依赖 DB 数据（用 H2 / Testcontainers）

### 🟡 P1 · 一个测试方法只测一个场景

不要在一个 `@Test` 里测 5 个 case。

---

## 十五、Git / PR 流程

### 🔴 P0 · 分支命名

```
feature/PROJ-1234-add-progressive-tiers
bugfix/PROJ-2345-fix-share-calc
hotfix/PROJ-3456-urgent-payment-fail
refactor/PROJ-4567-rebuild-event-handler
```

### 🔴 P0 · Commit Message 用 Conventional Commits

```
<type>(<scope>): <subject>

<body>

Refs: PROJ-1234
```

type：`feat / fix / refactor / perf / docs / style / test / chore`

scope：模块名（如 `bonus` / `share` / `payment`）

```
feat(bonus): 支持渐进式 CPA 阶梯多次入金累加

- 新增 ProgressiveBonusCalculator
- 调整 ClientSummaryDto 增加 cumulativeFtdAmount 字段
- 补充对应单测

Refs: PROJ-1234
```

### 🔴 P0 · PR 必须经过 Review 才能合并

- 至少 1 个研发 Reviewer 通过
- 涉及核心业务（如佣金计算）需 Tech Lead 通过
- 必须通过 CI 全部检查（编译、单测、SonarQube）

### 🟡 P1 · PR 描述必填

```markdown
## 背景
<Why>

## 改动
<What>

## 测试
- [ ] 单测
- [ ] 本地接口验证
- [ ] 联调

## 风险
<Risk and rollback plan>

## 影响范围
<Modules / 服务 / 数据>
```

### 🟢 P2 · 不在 PR 里夹带"顺手优化"

发现历史代码不规范？另开 PR 修，不要混在业务 PR 里。

---

## 十六、引用资料

- 《阿里巴巴 Java 开发手册（黄山版）》
- 《Effective Java》Joshua Bloch
- 《Clean Code》Robert C. Martin
- 公司 hytech-cpa 项目实践沉淀
- Spring 官方文档

---

## 最后：违反规范不是"末日"

发现自己以前写的代码不符合规范 —— **不要批量改**。按外科手术原则，只在改到那一行时顺手修复直接相关的违规，不要扩大化。

发现别人的代码不符合规范 —— Review 时**指出，不羞辱**。规范是工具，不是用来 PUA 队友的。
