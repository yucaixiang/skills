---
name: code-walkthrough
description: 跨项目代码理解助手 — 教 AI 系统化阅读、串联、解释任何 Java 后端代码。从 Controller / Service / Mapper 反向追踪到业务流程；识别设计模式；画时序图；定位代码所属业务模块。当测试 / 产品 / 新员工 / 跨岗位同事需要理解"这段代码做了什么 / 这个接口怎么调出来的 / 这个 bug 发生在哪一层"时调用。
version: 1.0.0
origin: SkillForge
---

# code-walkthrough · 跨项目代码理解助手

> 教 AI 怎么"读懂任何 Java 后端项目"，让非研发岗位（测试、产品、运营）也能自助理解代码。

---

## When to Activate

- 用户问"这段代码做了什么 / 为什么这么写"
- 用户想理解一个**接口 → 业务流程**的完整链路（"前端调 /xxx 接口最终走了哪些 Service？"）
- 用户想反向追踪一个错误：从异常 / 日志 / 告警出发，定位到代码源头
- 用户想理解某个类 / 方法的角色与上下文
- 新员工 / 测试 / 产品在 Claude Code 里说"这段代码什么意思"

> 跨项目通用，不绑定 hytech-cpa。但如果当前项目是 hytech-cpa，**优先级高于 `code-walkthrough` 的是 `hytech-cpa` Skill**（业务上下文更准）。

---

## 核心方法论：4 个阅读视角

```
┌──────────────────────────────────────────────────────────────┐
│  视角 1 · 业务视角                                            │
│    我在看的代码，对应的业务概念是什么？                       │
│    （账户管理 / 佣金计算 / 审批 ...）                         │
│                                                              │
│  视角 2 · 调用链视角                                          │
│    这段代码被谁调用？它又调用谁？                             │
│    （从 Controller → Service → Repository → Mapper）          │
│                                                              │
│  视角 3 · 数据视角                                            │
│    数据从哪里来？落到哪？经过了哪些状态？                     │
│    （Req → Entity → DTO → Res；DB 表与字段）                  │
│                                                              │
│  视角 4 · 设计视角                                            │
│    这段代码用了什么设计模式？为什么这么设计？                 │
│    （策略 / 状态机 / 模板方法 / 责任链 / 装饰）              │
└──────────────────────────────────────────────────────────────┘
```

---

## 标准阅读流程（7 步法）

### Step 1 · 项目鸟瞰（30 秒）

```bash
# 查看模块结构
ls -d */                        # 顶层目录
find . -name "pom.xml" -maxdepth 3   # 找 Maven 模块
cat README.md | head -100        # 读项目入口
```

**目标**：搞清楚有哪些模块，业务边界在哪。

### Step 2 · 找入口（按问题类型）

| 问题类型 | 找入口的方法 |
|---|---|
| "这个 URL 怎么实现" | `grep -rn "@RequestMapping.*xxx" --include="*.java"` |
| "这个错误怎么来的" | `grep -rn "{错误消息}" --include="*.java"` |
| "这个表怎么用" | `grep -rn "{table_name}" --include="*.xml" --include="*.java"` |
| "这个枚举谁在用" | `grep -rn "XxxEnum" --include="*.java"` |
| "这个定时任务做什么" | `grep -rn "@XxlJob.*xxx"` 或找 `*Job.java` |

### Step 3 · 顺藤摸瓜（自顶向下）

```
Controller (顶层入口)
    ↓ 找它调用的 Service
Service Interface
    ↓ 找它的 Impl
ServiceImpl
    ↓ 找它调用的 Repository / 外部 Feign / Service
Repository
    ↓ 找它调用的 Mapper
Mapper Interface
    ↓ 找对应的 Mapper.xml
SQL
```

### Step 4 · 关键变量追踪

读到方法里某个变量时，问：
- 它从哪传来的？
- 它在方法里被修改吗？
- 它传到哪去？

工具：IDE 的"Find Usages"或 `grep -n "varName"`。

### Step 5 · 画时序图（复杂场景）

复杂流程必须画图，否则光看代码会迷路。

**简化的 ASCII 时序图模板**：

```
Client       Controller    Service       Repository    Mapper      DB
  │              │            │              │            │         │
  ├─POST /xxx──→│             │              │            │         │
  │              ├──calc()──→ │              │            │         │
  │              │            ├─findById()→  │            │         │
  │              │            │              ├─select()──→│         │
  │              │            │              │            ├─SQL───→│
  │              │            │              │            │←──data─┤
  │              │            │              │←──Entity───┤         │
  │              │            │←──DTO────────┤            │         │
  │              │←──Res──────┤              │            │         │
  │←──JSON──────┤             │              │            │         │
```

### Step 6 · 识别模式

读到的代码用了哪些常见模式？识别后理解更快：

| 模式 | 代码特征 | 例 |
|---|---|---|
| **策略模式** | 接口 + 多个 Impl + `Map<Type, Impl>` 注册 | hytech-cpa 的 `BonusCalculator` |
| **模板方法** | 抽象类 + abstract 钩子方法 + 子类实现 | `AbstractBatchJob` |
| **责任链** | 处理器列表 + each.handle() 或 next.handle() | 校验链、过滤器链 |
| **状态机** | 状态枚举 + Action 枚举 + transition() | BPM `ApproveWork.workStatus` |
| **观察者 / 发布订阅** | `ApplicationEventPublisher` / Kafka 生产消费 | 用户同步事件 |
| **装饰器** | 同类型对象包装 + 增强 | Spring AOP / Filter |
| **SPI** | 接口 + ServiceLoader / Spring `getBeansOfType` | `ApproveWorkFinishedHandler` |

### Step 7 · 用自己的话讲一遍（验证理解）

读完代码，用 1 句话总结："这段代码做了 X，依赖 Y，影响 Z"。

讲不出来 = 没读懂。回 Step 1 重来。

---

## 实操技巧库

### 技巧 1 · 命名规律识别

读 Java 项目，从命名就能猜功能：

| 命名 | 角色 |
|---|---|
| `XxxController` | HTTP 入口 |
| `BsnXxxController` | 业务层 HTTP（hytech-cpa 约定） |
| `BckXxxController` | 数据层 HTTP（hytech-cpa 约定） |
| `XxxService` | 业务接口 |
| `XxxServiceImpl` | 业务实现 |
| `XxxCoreService` | 核心业务服务 |
| `XxxRepository` | 数据访问封装 |
| `XxxMapper` | MyBatis 映射接口 |
| `XxxEntity` | DB 表对应实体 |
| `XxxDto` | 层间传输对象 |
| `XxxReq` / `XxxRes` | API 入参 / 出参 |
| `XxxEnum` | 枚举 |
| `XxxConst` / `XxxConstant` | 常量 |
| `XxxConvert` | MapStruct 转换 |
| `XxxJob` | 定时任务 |
| `XxxHandler` | 事件 / 状态处理器 |
| `XxxFilter` | 过滤器 |
| `XxxAspect` | AOP 切面 |
| `XxxConfig` / `XxxConfiguration` | Spring 配置 |
| `XxxApi` | Feign 接口 |

### 技巧 2 · 注解快速识别

| 注解 | 含义 |
|---|---|
| `@RestController` `@Controller` | HTTP 入口类 |
| `@Service` | 业务 Bean |
| `@Repository` | 数据访问 Bean |
| `@Component` | 通用 Bean |
| `@Configuration` + `@Bean` | 配置类 |
| `@RequestMapping` `@GetMapping` `@PostMapping` | HTTP 路由 |
| `@RequestBody` | 从请求体取参 |
| `@PathVariable` | 路径参数 |
| `@RequestParam` | Query 参数 |
| `@Valid` `@Validated` | 参数校验 |
| `@Transactional` | 事务边界 |
| `@Async` | 异步执行 |
| `@Scheduled` `@XxlJob` | 定时任务 |
| `@KafkaListener` | Kafka 消费 |
| `@FeignClient` | 远程调用 |
| `@ConditionalOnXxx` | 条件装配 |
| `@Aspect` `@Around` `@Before` | AOP |

### 技巧 3 · 异常追溯

线上看到 `BizException: RULE_NOT_EXISTS`：

```bash
# 1. 找枚举定义
grep -rn "RULE_NOT_EXISTS" --include="*.java"
# → JobResponseEnum.java

# 2. 找谁抛出
grep -rn "RULE_NOT_EXISTS" --include="*.java" | grep -v Enum
# → BonusCalculationCoreServiceImpl.java:72

# 3. 看上下文
# 进入文件读上下 50 行，理解何时抛
```

### 技巧 4 · "这个表谁在写"

```bash
# 找所有写 tb_p_bonus_qftd 的地方
grep -rn "tb_p_bonus_qftd\|BonusQftd" --include="*.java" --include="*.xml"

# 写入操作通常关键字
grep -rn "INSERT.*tb_p_bonus_qftd\|insert.*BonusQftd" --include="*.xml" --include="*.java"
```

### 技巧 5 · "这个配置项在哪用"

```bash
# 找配置 cpa.alert.type 的使用方
grep -rn 'cpa.alert.type\|alert.type' --include="*.java" --include="*.yml"

# Spring @Value 用法
grep -rn '@Value("\${cpa\.alert' --include="*.java"
```

### 技巧 6 · 接口出入参快速画像

```java
@PostMapping("/page")
public InvokeResult<PageResult<CpaAccountRes>> page(
        @RequestBody @Valid CpaAccountPageReq req) { ... }
```

3 秒看懂：
- HTTP POST `/page`
- 入参：JSON Body，类型 `CpaAccountPageReq`，强校验
- 出参：标准 `InvokeResult<PageResult<CpaAccountRes>>`
- 进 `CpaAccountPageReq` 看具体字段

---

## 跨岗位场景

### 场景 1 · 测试：「为什么这个订单撮合失败」

```
1. 在日志里搜该订单 → 拿到 clientUserId
2. MDC eventid 串起整条链路
3. 找到调用链中第一条 ERROR
4. 用 grep 找异常消息源头 → 锁定具体方法
5. 读该方法上下文，理解失败原因
6. 给研发提单时附上：日志片段 + 代码位置 + 推断原因
```

### 场景 2 · 产品：「这个功能现在怎么实现的」

```
1. 找前端调用的接口 URL
2. 后端搜 @RequestMapping("/xxx")
3. 从 Controller → Service 串到业务规则
4. 用自然语言写下"它的流程是 X → Y → Z"
5. 跟 PRD 比对：有没有偏差？
```

### 场景 3 · 运维：「这个告警从哪里发的」

```
1. 复制告警 title 到代码搜索
   grep -rn "title.*\"佣金计算\"" --include="*.java"
2. 找到 larkCommMessagePusher.sendAlert(...) 调用点
3. 看上下文：什么场景触发的？
4. 结合告警时的 eventid 看日志
```

### 场景 4 · 新员工：「这个项目从哪开始读」

```
1. README.md → 项目定位
2. pom.xml → 模块清单
3. *.java 找带 main 方法的 → 启动入口
4. Application 类 → @SpringBootApplication 扫描包
5. 找几个 Controller 看典型接口
6. 按本 Skill 的 7 步法逐个深入
```

---

## 输出范式：怎么回答"这段代码做什么"

### 范式 1 · 简短回答（1-2 句话）

```
{Class}.{method} 是 {域} 模块的 {角色}，负责 {一句话目的}。
被 {主要调用方} 调用，依赖 {主要下游}。
```

例：
> `BonusCalculationCoreServiceImpl.calculateQftdBonus` 是 hytech-cpa 佣金域的核心调度服务，负责按佣金类型路由到具体计算器。被 `EventHandleCoreService` 调用，依赖 4 个 `BonusCalculator` 实现。

### 范式 2 · 中等回答（含调用链）

```
{Class}.{method} 做什么：{业务目的}

调用链：
  上游：{caller1}, {caller2}
  下游：{callee1} → {callee2} → {DB/MQ}

关键逻辑：
  1. {步骤 1}
  2. {步骤 2}
  3. {步骤 3}

异常路径：
  - {场景}：{处理}
```

### 范式 3 · 完整回答（含时序图 + 反例）

含上面所有 + 时序图（ASCII） + 容易踩坑的地方 + 联调建议。

---

## 反例：常见误读

| 误读 | 实际 | 教训 |
|---|---|---|
| 看到 `@Async` 以为同步 | 异步执行，调用方立刻返回 | 注意结果如何回调 |
| 看到 `@Transactional` 以为一定生效 | 同类自调用无效 | 看是不是被 Spring 代理过 |
| 看到 `Optional` 没看 isPresent | 可能 NPE | 必 `.orElse(...)` 或 `.orElseThrow(...)` |
| 看到 `static` 字段以为线程不安全 | 不可变就安全 | 关注是否 final + 不可变类型 |
| 看到 `synchronized` 以为分布式锁 | 只是单机 JVM 锁 | 分布式必须 Redisson |
| 看到 `try-catch (Exception e)` 以为捕获了一切 | 仍可能漏 Throwable 子类（OOM/StackOverflow） | 系统级异常通常不该 catch |

---

## 工具速查

### `grep` 黄金组合

```bash
# 含上下文
grep -rn "xxx" --include="*.java" -A 5 -B 2

# 多关键词
grep -rn "xxx\|yyy" --include="*.java"

# 排除 test
grep -rn "xxx" --include="*.java" | grep -v "/test/"

# 文件级统计
grep -rl "xxx" --include="*.java" | wc -l
```

### `find` 找文件

```bash
# 找所有 *Job.java
find . -name "*Job.java" -not -path "*/target/*"

# 找带 main 方法的类（启动类）
grep -rln "public static void main" --include="*.java"
```

### IDE 操作（IntelliJ）

| 操作 | 快捷键 (macOS) |
|---|---|
| Find Usages | `⌥ F7` |
| Go to Implementation | `⌘ ⌥ B` |
| Go to Declaration | `⌘ B` |
| Recent Files | `⌘ E` |
| Search Everywhere | `Shift Shift` |
| Find in Files | `⌘ Shift F` |
| Structure（类大纲） | `⌘ F12` |

---

## 配合其他 Skill 的工作流

```
[需要懂业务] → code-walkthrough（顺藤摸瓜） + hytech-cpa（业务上下文）
                              ↓
[准备改代码] → dev-standards（规范）
                              ↓
[写完了改动] → code-review-rules（自检）
                              ↓
[需要写测试] → test-case-generator + api-test-automation
                              ↓
[需要发飞书] → lark-cli（同步飞书文档 / 发周报）
```

---

## 给非研发同学的话

- 不会写代码也能**读代码**。代码本身有大量自解释能力（命名 / 注释 / 类型）
- 读不懂是正常的。**反复 3 次**：第一次混乱，第二次清晰，第三次内化
- 不要怕问"傻问题"。AI 永远耐心，远比同事好
- 读懂一个项目 = 拥有这个项目的话语权

---

## 引用

- 《重构（第 2 版）》Martin Fowler — 怎么识别坏味道
- 《设计模式》GoF
- IntelliJ IDEA 官方文档 — Navigation 章节
