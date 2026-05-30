# 05 · 开发指南

> 本文档是 hytech-cpa 的"开发手册"，包含包结构、命名规范、Controller/Service/Mapper 标准写法、提交规约等。
> **写任何代码前必读。**

---

## 一、包结构（强制）

每个服务的标准包结构：

```
com.hytech.cpa.{domain}.{service-type}.
├── config/         配置类（@Configuration、@Bean 定义）
├── constant/       静态常量类
├── controller/     控制层（Controller）
├── enums/          枚举类
├── jobs/           XxlJob 任务类（仅 job-executor）
├── mapper/         MyBatis Mapper 接口
├── model/
│   ├── req/        API 入参类，类名以 Req 结尾
│   ├── res/        API 出参类，类名以 Res 结尾
│   ├── db/         DB 实体类，类名为表名驼峰
│   └── dto/        数据传输类，类名以 DTO 结尾
├── service/        服务接口
│   └── impl/       服务实现
└── util/           工具类
```

`{domain}` ∈ `admin / client / job / open`
`{service-type}` ∈ `application / business / backend`

---

## 二、命名规范

### 2.1 类命名

| 角色 | 命名规则 | 示例 |
|---|---|---|
| Controller（business） | `Bsn` + 业务名 + `Controller` | `BsnCpaAccountController` |
| Controller（backend） | `Bck` + 业务名 + `Controller` | `BckCpaAccountController` |
| Service 接口 | 业务名 + `Service` | `CpaAccountService` |
| Service 实现 | 业务名 + `ServiceImpl` | `CpaAccountServiceImpl` |
| Core Service | 业务名 + `CoreService` | `BonusCalculationCoreService` |
| Mapper | 业务名 + `Mapper` | `CpaAccountMapper` |
| Repository | 业务名 + `Repository` | `CpaAccountRepository` |
| Entity | 表名驼峰 + `Entity` | `CpaAccountEntity` |
| DTO | 业务名 + `Dto` | `ClientSummaryDto` |
| Req 入参 | 业务名 + `Req` | `CpaAccountPageReq` |
| Res 出参 | 业务名 + `Res` | `CpaAccountListRes` |
| 枚举 | 业务名 + `Enum` | `CpaAccountTypeEnum` |
| 异常 | 业务名 + `Exception` | （通用用 `BizException`） |
| Feign Api | 服务名 + `Api` | `PlatformJobExternalApi` |
| Job | 业务名 + `Job` | `BonusBatchJob` |
| Convert | 业务名 + `Convert` | `ClientSummaryConvert` |

### 2.2 字段命名

- Java：camelCase（如 `cpaAccount`、`bonusAmount`）
- DB：snake_case（如 `cpa_account`、`bonus_amount`）
- MyBatis 自动驼峰映射（确保配置 `mapUnderscoreToCamelCase: true`）
- 常量：`UPPER_SNAKE_CASE`（如 `MAX_RETRY_COUNT`）

### 2.3 URL 命名

- business 接口：`/bsn/{业务}/{操作}`，如 `/bsn/cpa-account/page`
- backend 接口：`/bck/{业务}/{操作}`，如 `/bck/cpa-account/query-by-id`
- 业务名用 kebab-case（短横线）
- 操作动词：`page`/`list`/`query`/`create`/`update`/`delete`/`export` 等

---

## 三、Controller 标准写法

```java
@Slf4j
@Tag(name = "代理账户管理")
@RestController
@RequestMapping("/bsn/cpa-account")
@RequiredArgsConstructor
public class BsnCpaAccountController {

    private final CpaAccountService cpaAccountService;

    @Operation(summary = "代理账户分页查询")
    @PostMapping("/page")
    public InvokeResult<PageResult<CpaAccountRes>> page(
            @RequestBody @Valid CpaAccountPageReq req) {
        log.info("page query req:{}", JsonUtil.toJsonString(req));
        return InvokeResult.success(cpaAccountService.page(req));
    }

    @Operation(summary = "代理账户详情")
    @GetMapping("/{account}")
    public InvokeResult<CpaAccountRes> detail(@PathVariable Long account) {
        return InvokeResult.success(cpaAccountService.detail(account));
    }
}
```

**关键点**：
- Controller **不写业务逻辑**，仅协议转换与编排
- 入参强校验：`@RequestBody @Valid`
- 出参统一封装：`InvokeResult<T>`
- 用 `@RequiredArgsConstructor` 构造注入（推荐）
- 关键操作打 info 日志

---

## 四、Service 标准写法

### 4.1 接口

```java
public interface CpaAccountService {

    PageResult<CpaAccountRes> page(CpaAccountPageReq req);

    CpaAccountRes detail(Long account);

    void upgrade(Long account, String targetType);
}
```

### 4.2 实现

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class CpaAccountServiceImpl implements CpaAccountService {

    private final CpaAccountRepository cpaAccountRepository;
    private final CpaAccountInternalApi cpaAccountInternalApi;  // 同域 Feign
    private final PlatformUserExternalApi userExternalApi;       // 跨域 Feign
    private final RedissonLock redissonLock;
    private final CommMessagePusher larkCommMessagePusher;

    @Override
    public PageResult<CpaAccountRes> page(CpaAccountPageReq req) {
        log.info("page query, req:{}", JsonUtil.toJsonString(req));

        // 1. 调 backend 取数据
        PageResult<CpaAccountDto> dtoPage = cpaAccountInternalApi.page(req).getOrThrow();

        // 2. 补充外部数据（如用户姓名）
        List<Long> userIds = dtoPage.getRecords().stream()
            .map(CpaAccountDto::getUserId).toList();
        Map<Long, DapUserInfoResp> userMap = loadUserMap(userIds);

        // 3. 组装 Res
        List<CpaAccountRes> records = dtoPage.getRecords().stream()
            .map(dto -> assembleRes(dto, userMap.get(dto.getUserId())))
            .toList();

        return PageResult.of(records, dtoPage.getTotal());
    }

    private Map<Long, DapUserInfoResp> loadUserMap(List<Long> userIds) {
        if (CollectionUtils.isEmpty(userIds)) return Collections.emptyMap();
        return userExternalApi.queryUserInfoList(userIds.stream()
                .map(Long::intValue).toList())
            .getOrThrow().stream()
            .collect(Collectors.toMap(DapUserInfoResp::getUserId, Function.identity()));
    }
}
```

**关键点**：
- 实现类用 `@Service` 注解
- 注入用 `@RequiredArgsConstructor` + `final` 字段
- 复杂方法拆小（每个方法 < 50 行）
- 调 Feign 用 `.getOrThrow()` 而不是裸 `getData()`
- 跨业务数据通过外部 Feign 取，禁止直连其他业务库

---

## 五、Mapper / Repository 标准写法

### 5.1 Repository（推荐封装层）

```java
@Repository
@RequiredArgsConstructor
public class CpaAccountRepository {

    private final CpaAccountMapper cpaAccountMapper;

    public CpaAccountEntity findByAccount(Long account) {
        return cpaAccountMapper.selectOne(new LambdaQueryWrapper<CpaAccountEntity>()
            .eq(CpaAccountEntity::getAccount, account));
    }

    public List<CpaAccountEntity> findByUserIds(List<Long> userIds) {
        if (CollectionUtils.isEmpty(userIds)) return Collections.emptyList();
        return cpaAccountMapper.selectList(new LambdaQueryWrapper<CpaAccountEntity>()
            .in(CpaAccountEntity::getUserId, userIds));
    }

    public void insertBatch(List<CpaAccountEntity> entities) {
        cpaAccountMapper.insertBatch(entities);
    }
}
```

### 5.2 Mapper（与 XML 配对）

```java
@Mapper
public interface CpaAccountMapper extends BaseMapper<CpaAccountEntity> {

    int insertBatch(@Param("list") List<CpaAccountEntity> list);

    List<CpaAccountStatRes> statByDate(@Param("startDate") LocalDate startDate,
                                        @Param("endDate") LocalDate endDate);
}
```

XML 位置：`src/main/resources/mapper/CpaAccountMapper.xml`

### 5.3 关键点

- **Service 不直接调 Mapper**，必须经过 Repository
- Repository 不返回 PO 给上层调用（如果是 backend 给 business 用，必须先 Convert 为 DTO）
- 复杂查询用 XML，简单 CRUD 用 `BaseMapper`
- 批量操作必须批量 SQL，禁止 `for + insert`

---

## 六、统一异常处理

### 6.1 抛业务异常

```java
import com.hytech.common.core.exception.biz.BizException;

if (cpaAccount == null) {
    throw new BizException(AdminResponseEnum.CPA_ACCOUNT_NOT_FOUND);
}
```

### 6.2 ResponseEnum 定义

每个领域有自己的 Response 枚举：
- `AdminResponseEnum` · ToB
- `ClientResponseEnum` · ToC
- `JobResponseEnum` · ToJ
- `OpenResponseEnum` · ToO
- `BackendResponseEnum` · backend 层

### 6.3 全局异常处理

由 `GlobalExceptionHandler` 拦截，返回标准 `InvokeResult`：

```java
{
  "code": 4001,
  "message": "代理账户不存在",
  "data": null
}
```

业务代码**只管抛异常**，不要 try-catch 后返回错误结果。

---

## 七、事务规范

### 7.1 标准方式

```java
@Transactional(rollbackFor = Exception.class)
public void completeBonus(BonusQftdDto dto) {
    bonusQftdRepository.insert(...);
    settlementRepository.insert(...);
    clientSummaryRepository.update(...);
}
```

### 7.2 跨服务事务

跨 Feign 调用**不支持分布式事务**，需采用：
- 最大努力通知（Kafka + 补偿任务）
- 本地消息表
- 业务补偿 Job

### 7.3 锁 + 事务的顺序

```java
redissonLock.lock(key);
try {
    // 锁内开启事务
    bonusTransactional.completeBonus(...);  // @Transactional 在这个方法上
} finally {
    redissonLock.unlock(key);
}
```

**注意**：`@Transactional` 自调用无效（Spring AOP 限制），所以 `bonusTransactional` 是独立 Bean。

---

## 八、Entity ↔ DTO 转换

强制用 **MapStruct**：

```java
@Mapper(componentModel = "spring",
        unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface CpaAccountConvert {
    CpaAccountConvert INSTANCE = Mappers.getMapper(CpaAccountConvert.class);

    CpaAccountDto entityToDto(CpaAccountEntity entity);
    CpaAccountEntity dtoToEntity(CpaAccountDto dto);
    List<CpaAccountDto> entitiesToDtos(List<CpaAccountEntity> entities);
}
```

使用：

```java
CpaAccountDto dto = CpaAccountConvert.INSTANCE.entityToDto(entity);
```

**禁止** 手写 setter 一个个赋值。

---

## 九、依赖注入规范

### 9.1 推荐：构造注入

```java
@Service
@RequiredArgsConstructor
public class XxxService {
    private final XxxRepository xxxRepository;
    private final YyyService yyyService;
}
```

### 9.2 可接受：@Resource

```java
@Service
public class XxxService {
    @Resource
    private XxxRepository xxxRepository;
}
```

### 9.3 避免：@Autowired

旧代码可见，新写禁止。理由：构造注入更便于单测且支持 final。

---

## 十、Lombok 使用规范

| 注解 | 用途 | 备注 |
|---|---|---|
| `@Getter @Setter` | 替代手写 getter/setter | Entity 必备 |
| `@SuperBuilder` | 继承场景的 Builder | Entity 继承 BaseEntity 用这个 |
| `@Builder` | 普通 Builder | DTO/Req/Res 用这个 |
| `@AllArgsConstructor` | 全参构造 | Entity 必备 |
| `@NoArgsConstructor` | 无参构造 | Entity 必备（MyBatis 反射用） |
| `@ToString(callSuper = true)` | toString | Entity 用 |
| `@EqualsAndHashCode(callSuper = true)` | equals/hashCode | Entity 用 |
| `@RequiredArgsConstructor` | final 字段的构造注入 | Service/Controller 用 |
| `@Slf4j` | 注入 logger | 所有需要日志的类 |
| `@Data` | Getter+Setter+ToString | DTO/Req/Res 用 |

---

## 十一、JSON 序列化

### 11.1 优先 Jackson

```java
JsonUtil.toJsonString(obj);                          // 序列化
JsonUtil.parseObject(json, MyClass.class);            // 反序列化
JsonUtil.parseArray(json, new TypeReference<...>() {}); // List 反序列化
```

### 11.2 日期格式

```java
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime registerTime;

@JsonFormat(pattern = "yyyy-MM-dd")
private LocalDate rangeDateStart;
```

### 11.3 BigDecimal

```java
@JsonSerialize(using = ToStringSerializer.class)
private BigDecimal bonusAmount;
```

避免前端精度丢失。

---

## 十二、单元测试

### 12.1 命名

- 类名：被测类 + `Test`，如 `CpaAccountServiceImplTest`
- 方法名：`should_xxx_when_xxx`

### 12.2 工具

- JUnit 5
- Mockito
- Spring Boot Test

### 12.3 重点测什么

- ✅ Service 的业务逻辑（边界、异常、状态转换）
- ✅ BonusCalculator 各 case
- ✅ 复杂 SQL 的 Mapper（用 H2 / @MybatisTest）
- ❌ 不必要：纯 getter/setter、单纯 Convert 转换

---

## 十三、Git 提交规约

### 13.1 Commit Message 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 13.2 type

| type | 含义 |
|---|---|
| `feat` | 新功能 |
| `fix` | 修 bug |
| `refactor` | 重构（无新功能/无 bug） |
| `perf` | 性能优化 |
| `docs` | 文档 |
| `style` | 格式（无代码逻辑变更） |
| `test` | 测试 |
| `chore` | 构建/工具/依赖 |

### 13.3 scope（按模块）

`admin / client / job / open / common / alert / bpm / core` 等

### 13.4 示例

```
feat(bonus): 支持渐进式 CPA 阶梯多次入金累加

- 新增 ProgressiveBonusCalculator 实现类
- 调整 ClientSummaryDto 增加 cumulativeFtdAmount 字段
- 补充对应单测

Refs: PROJ-1234
```

---

## 十四、Code Review Checklist

提交 PR 前自查：

- [ ] 分层正确（没有 business 访问其他业务数据库等）
- [ ] URL 前缀正确（`/bsn` 或 `/bck`）
- [ ] 没有把 Entity 作为接口出入参
- [ ] Redis Key / Kafka Topic 用常量，符合命名规范
- [ ] 复杂逻辑有日志（带业务标识）
- [ ] 异常用 BizException + ResponseEnum，未裸 try-catch 吞异常
- [ ] 关键业务用 RedissonLock 加锁
- [ ] 写库操作有事务 + 写流水（如余额变动）
- [ ] 触发告警的失败场景用 `larkCommMessagePusher.sendAlert`
- [ ] 关联表更新一致（如关系 path、valid 字段）
- [ ] 关键方法补充了单测
- [ ] 没有提交日志/敏感配置

---

## 十五、本地启动指南

### 15.1 前置依赖

- Java 17
- Maven 3.8+
- MySQL 8（业务库）
- Redis 7
- Consul（可用 Docker 起一个）
- Kafka（可用 Docker 起一个）

### 15.2 启动顺序

```
1. Consul → 2. MySQL → 3. Redis → 4. Kafka
5. CpaAdminApplication（9101）
6. CpaClientApplication（9103）
7. CpaJobApplication（9105）
8. CpaOpenApplication（9106）
```

### 15.3 启动参数

参考 README.md 中各 Application 的启动参数。

### 15.4 接口文档

启动后访问对应 `/doc.html`。

### 15.5 Mock 外部依赖

DAP/User、MT4/MT5 等外部数据源，本地开发用：
- `application-local.yml` 切到 mock 数据源
- 或用 `hytech-cpa-cellxpert-api-test` 提供的 mock 工具
