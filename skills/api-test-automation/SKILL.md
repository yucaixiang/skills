---
name: api-test-automation
description: 后端 API 自动化测试脚本生成 Skill — 基于测试用例（来自 test-case-generator）或需求文档，自动生成 JUnit 5 + RestAssured 风格的 Java API 测试脚本，支持鉴权、数据驱动、断言、CI 集成。覆盖 hytech-cpa 等 Spring Boot 项目。当需要"把用例落地为脚本 / 写接口自动化 / 做接口回归"时调用。
version: 1.0.0
origin: SkillForge
---

# api-test-automation · API 自动化测试脚本生成

> 把测试用例**直接变成可跑的 Java 测试代码**。配合 `test-case-generator` 形成"设计 → 落地 → 执行"完整闭环。

---

## When to Activate

- 用户说"把这批用例转成脚本"
- 用户说"为 xx 接口写自动化"
- 用户说"加一组接口回归测试"
- 与 `test-case-generator` Skill 串联（先设计，再脚本化）
- CI 中需要自动跑接口回归

---

## 技术栈

| 组件 | 用途 |
|---|---|
| **JUnit 5** | 测试框架 |
| **RestAssured** | HTTP 测试 DSL |
| **AssertJ** | 流式断言 |
| **WireMock**（可选） | mock 外部依赖 |
| **Testcontainers**（可选） | 起 MySQL / Redis 等真实组件 |
| **Allure** / **Surefire Report** | 测试报告 |
| **Maven / Gradle** | 构建与运行 |

理由：跟 hytech-cpa 项目同栈（Java + Spring Boot），研发也能维护。

---

## 标准项目骨架

```
hytech-cpa-api-test/
├── pom.xml                              # 依赖配置
├── src/main/java/com/hytech/cpa/test/
│   ├── client/                          # 通用 API 调用客户端
│   │   ├── ApiClient.java               # RestAssured 包装
│   │   └── AuthClient.java              # 鉴权与 Token 管理
│   ├── config/
│   │   ├── TestConfig.java              # 测试环境配置
│   │   └── DataSourceConfig.java        # 测试 DB 配置（可选）
│   └── util/
│       ├── DataFactory.java             # 测试数据工厂
│       └── DbCleaner.java               # 测试数据清理
└── src/test/java/com/hytech/cpa/test/
    ├── bonus/
    │   ├── BonusCalculateApiTest.java   # 佣金计算 API 测试
    │   └── BonusShareApiTest.java
    ├── account/
    │   └── CpaAccountApiTest.java
    └── ApiTestBase.java                 # 测试基类（鉴权 / setup / teardown）
```

---

## Step 1 · 标准依赖（pom.xml）

```xml
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>

    <!-- RestAssured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>json-path</artifactId>
        <version>5.4.0</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.1</version>
        <scope>test</scope>
    </dependency>

    <!-- Lombok（测试也用） -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>

    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.16.0</version>
    </dependency>

    <!-- Allure（可选）-->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-rest-assured</artifactId>
        <version>2.25.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <includes>
                    <include>**/*ApiTest.java</include>
                </includes>
                <systemPropertyVariables>
                    <test.env>${test.env}</test.env>
                </systemPropertyVariables>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## Step 2 · 测试基类（鉴权 + 配置）

```java
package com.hytech.cpa.test;

import io.restassured.RestAssured;
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import org.junit.jupiter.api.BeforeAll;

import static io.restassured.RestAssured.given;

/**
 * 所有 API 测试的基类。
 * 负责：环境配置、鉴权、通用 RequestSpec。
 */
public abstract class ApiTestBase {

    protected static RequestSpecification adminSpec;
    protected static RequestSpecification clientSpec;
    protected static String adminToken;
    protected static String clientToken;

    @BeforeAll
    static void setUp() {
        String env = System.getProperty("test.env", "dev");
        String baseUrl = loadBaseUrl(env);
        RestAssured.baseURI = baseUrl;
        RestAssured.basePath = "/api";

        // 初始化 admin 鉴权
        adminToken = login("admin@example.com", "test123!");
        adminSpec = new RequestSpecBuilder()
                .setContentType(ContentType.JSON)
                .addHeader("Authorization", "Bearer " + adminToken)
                .build();

        // 初始化 client（代理）鉴权
        clientToken = login("agent@example.com", "test123!");
        clientSpec = new RequestSpecBuilder()
                .setContentType(ContentType.JSON)
                .addHeader("Authorization", "Bearer " + clientToken)
                .build();
    }

    private static String loadBaseUrl(String env) {
        return switch (env) {
            case "dev"  -> "http://localhost:9101";
            case "test" -> "http://test.cpa.internal:9101";
            case "uat"  -> "http://uat.cpa.internal:9101";
            default     -> throw new IllegalArgumentException("unknown env: " + env);
        };
    }

    private static String login(String username, String password) {
        return given()
                .contentType(ContentType.JSON)
                .body(Map.of("username", username, "password", password))
                .post("/auth/login")
                .then().statusCode(200)
                .extract().path("data.token");
    }
}
```

---

## Step 3 · 接口测试标准模板

### 模板 A · 简单 GET 接口

```java
package com.hytech.cpa.test.account;

import com.hytech.cpa.test.ApiTestBase;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

class CpaAccountApiTest extends ApiTestBase {

    @Test
    void should_return_account_detail_when_account_exists() {
        Long existingAccount = 1001L;

        given()
            .spec(adminSpec)
            .pathParam("account", existingAccount)
        .when()
            .get("/bsn/cpa-account/{account}")
        .then()
            .statusCode(200)
            .body("code", equalTo(0))
            .body("data.account", equalTo(existingAccount.intValue()))
            .body("data.accountType", isOneOf("CPA", "IB", "AFFILIATE"))
            .body("data.accountStatus", is("VALID"));
    }

    @Test
    void should_return_not_found_when_account_not_exists() {
        Long nonExistAccount = 99999999L;

        given()
            .spec(adminSpec)
            .pathParam("account", nonExistAccount)
        .when()
            .get("/bsn/cpa-account/{account}")
        .then()
            .statusCode(200)  // 业务异常通常仍 200，看 code 字段
            .body("code", equalTo(4001))
            .body("message", containsString("代理账户不存在"));
    }
}
```

### 模板 B · POST 分页查询

```java
@Test
void should_paginate_accounts_correctly() {
    Map<String, Object> req = Map.of(
        "page", 1,
        "pageSize", 10,
        "accountStatus", "VALID"
    );

    given()
        .spec(adminSpec)
        .body(req)
    .when()
        .post("/bsn/cpa-account/page")
    .then()
        .statusCode(200)
        .body("code", equalTo(0))
        .body("data.records", hasSize(lessThanOrEqualTo(10)))
        .body("data.total", greaterThanOrEqualTo(0))
        .body("data.records[0].accountStatus", equalTo("VALID"));
}
```

### 模板 C · 参数校验失败

```java
@Test
void should_return_400_when_page_size_exceeds_max() {
    Map<String, Object> req = Map.of(
        "page", 1,
        "pageSize", 10000  // 超过最大允许值
    );

    given()
        .spec(adminSpec)
        .body(req)
    .when()
        .post("/bsn/cpa-account/page")
    .then()
        .statusCode(400)
        .body("message", containsString("pageSize"));
}
```

### 模板 D · 鉴权失败

```java
@Test
void should_return_401_when_no_token() {
    given()
        .contentType("application/json")
        // 不带 token
    .when()
        .post("/bsn/cpa-account/page")
    .then()
        .statusCode(401);
}

@Test
void should_return_403_when_client_token_access_admin_api() {
    given()
        .spec(clientSpec)  // 用 client token 调 admin 接口
    .when()
        .post("/bsn/cpa-account/page")
    .then()
        .statusCode(403);
}
```

### 模板 E · 业务流程（多接口串联）

```java
@Test
void should_complete_full_payment_flow() {
    // Step 1 · 代理申请出金
    Long applyId = given()
        .spec(clientSpec)
        .body(Map.of("payAmount", "500.00", "currency", "USD"))
    .when()
        .post("/bsn/payment/apply")
    .then()
        .statusCode(200)
        .body("code", equalTo(0))
        .extract().path("data.applyId");

    // Step 2 · 验证申请状态为审批中
    given()
        .spec(adminSpec)
        .pathParam("id", applyId)
    .when()
        .get("/bsn/payment/{id}")
    .then()
        .body("data.applyStatus", equalTo("P"));

    // Step 3 · 管理员审批通过
    given()
        .spec(adminSpec)
        .body(Map.of("applyId", applyId, "action", "PASS", "opinion", "approved"))
    .when()
        .post("/bsn/bonus-approve/decide")
    .then()
        .statusCode(200)
        .body("code", equalTo(0));

    // Step 4 · 验证状态变为 S
    given()
        .spec(adminSpec)
        .pathParam("id", applyId)
    .when()
        .get("/bsn/payment/{id}")
    .then()
        .body("data.applyStatus", equalTo("S"));
}
```

### 模板 F · 数据驱动测试（参数化）

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

@ParameterizedTest(name = "ftdAmount={0} should hit bonusItemId={1}")
@CsvSource({
    "100, 10, 'FTD 区间 [100, 500) 命中档位 1'",
    "200, 10, 'FTD 区间 [100, 500) 命中档位 1'",
    "499, 10, 'FTD 区间右边界 -1 仍在档位 1'",
    "500, 11, 'FTD 区间左边界进入档位 2'",
    "1000, 12, 'FTD 区间进入档位 3'",
    "99, null, 'FTD 不合格'"
})
void should_calculate_correct_ftd_bonus(int ftdAmount, String expectedItemId, String caseDesc) {
    Long clientUserId = createTestClient(ftdAmount);

    // 触发计算
    given().spec(adminSpec)
        .body(Map.of("clientUserId", clientUserId))
    .when()
        .post("/bsn/bonus/recalculate")
    .then().statusCode(200);

    // 验证结果
    given().spec(adminSpec)
        .queryParam("clientUserId", clientUserId)
    .when()
        .get("/bsn/bonus/qftd")
    .then()
        .body("data.bonusItemId", expectedItemId.equals("null") ? nullValue() : equalTo(Integer.parseInt(expectedItemId)));
}
```

### 模板 G · 异步业务等待

```java
import org.awaitility.Awaitility;
import java.time.Duration;

@Test
void should_generate_bonus_within_30s_after_payment_event() {
    Long clientUserId = setupClientWithSignedAgreement();

    // 触发入金事件（模拟 MT4 数据）
    insertPaymentEvent(clientUserId, new BigDecimal("200.00"));

    // 等待异步处理（最多 30s）
    Awaitility.await()
        .atMost(Duration.ofSeconds(30))
        .pollInterval(Duration.ofSeconds(2))
        .untilAsserted(() -> {
            given().spec(adminSpec)
                .queryParam("clientUserId", clientUserId)
            .when()
                .get("/bsn/bonus/qftd")
            .then()
                .body("data", not(empty()));
        });
}
```

---

## Step 4 · 测试数据管理

### A · DataFactory 工厂

```java
package com.hytech.cpa.test.util;

import lombok.experimental.UtilityClass;

@UtilityClass
public class DataFactory {

    public static Map<String, Object> createCpaAccountReq(String type) {
        return Map.of(
            "accountType", type,
            "name", "TestAgent_" + System.currentTimeMillis(),
            "email", "test_" + System.currentTimeMillis() + "@example.com",
            "country", "AU"
        );
    }

    public static Long createAndReturnId(RequestSpecification spec, String url, Map<String, Object> req) {
        return given().spec(spec).body(req)
            .when().post(url)
            .then().statusCode(200).body("code", equalTo(0))
            .extract().path("data.id");
    }
}
```

### B · 数据清理（@AfterEach / @AfterAll）

```java
@AfterEach
void cleanupTestData() {
    if (createdAccountId != null) {
        given().spec(adminSpec)
            .pathParam("id", createdAccountId)
        .when().delete("/bsn/cpa-account/{id}")
        .then().statusCode(200);
    }
}
```

### C · 测试 DB（极端场景）

直接连测试库做断言（仅在必要时）：

```java
@Autowired
private JdbcTemplate jdbcTemplate;

@Test
void should_create_balance_record_after_payment_approved() {
    Long applyId = doFullPaymentFlow();
    Thread.sleep(2000);  // 等异步入库

    Integer recordCount = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM tb_p_c_account_balance_record WHERE apply_id = ?",
        Integer.class, applyId);

    assertThat(recordCount).isEqualTo(1);
}
```

---

## Step 5 · CI 集成

### `.gitlab-ci.yml` 示例

```yaml
stages:
  - test
  - report

api-regression:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - cd hytech-cpa-api-test
    - mvn test -Dtest.env=test
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/*.xml
    paths:
      - target/allure-results/
  only:
    - merge_requests
    - main

allure-report:
  stage: report
  image: frankescobar/allure-docker-service
  script:
    - allure generate target/allure-results -o public/allure --clean
  artifacts:
    paths:
      - public/
  only:
    - main
```

### Jenkins Pipeline 示例

```groovy
pipeline {
    agent any
    stages {
        stage('API Test') {
            steps {
                dir('hytech-cpa-api-test') {
                    sh 'mvn test -Dtest.env=${TARGET_ENV}'
                }
            }
            post {
                always {
                    junit 'hytech-cpa-api-test/target/surefire-reports/*.xml'
                    allure includeProperties: false,
                           jdk: '',
                           results: [[path: 'hytech-cpa-api-test/target/allure-results']]
                }
            }
        }
    }
    post {
        failure {
            larkSendAlert(title: '接口回归失败',
                          message: "build #${BUILD_NUMBER} 失败：${BUILD_URL}")
        }
    }
}
```

---

## Step 6 · 与 SkillForge / lark-cli 集成

接口测试失败时自动发飞书告警（通过 `lark-cli`）：

```bash
lark-cli im send \
  --chat "#质量保障群" \
  --content "🚨 接口回归失败：${BUILD_URL}，请关注"
```

测试报告链接同步到飞书文档：

```bash
lark-cli docs +update --api-version v2 \
  --doc "https://x.larksuite.com/wiki/QualityReport" \
  --command append \
  --content "<p>${BUILD_TAG} 测试报告：${ALLURE_URL}</p>"
```

---

## 与 test-case-generator 的接驳

`test-case-generator` 输出的用例格式：

```json
{
  "caseId": "CASE-TRANSFER-001",
  "method": "POST",
  "url": "/bsn/transfer-client/batch",
  "auth": "admin",
  "body": { ... },
  "expected": {
    "statusCode": 200,
    "code": 0,
    "asserts": [
      "data.success_count == 1",
      "data.fail_count == 0"
    ]
  },
  "preCondition": [ ... ],
  "postCleanup": [ ... ]
}
```

`api-test-automation` 可以**直接读这个 JSON 生成对应的 JUnit 方法**：

```java
@Test
@DisplayName("CASE-TRANSFER-001 · 单条批量绑定成功")
void case_transfer_001() {
    // 前置条件
    setupAccount(1001);
    setupClientBoundTo(100001, 999);

    // 调用接口
    Map<String, Object> body = Map.of(
        "rows", List.of(Map.of("cpaAccount", 1001, "clientUserIds", List.of(100001)))
    );

    given().spec(adminSpec).body(body)
    .when().post("/bsn/transfer-client/batch")
    .then()
        .statusCode(200)
        .body("code", equalTo(0))
        .body("data.success_count", equalTo(1))
        .body("data.fail_count", equalTo(0));

    // 数据校验
    assertRelationValid(100001, 999, 0);
    assertRelationValid(100001, 1001, 1);
}
```

---

## 最佳实践

### 必做

- ✅ 测试方法名 `should_xxx_when_xxx`，IDE 自动展开能讲清楚 case
- ✅ 一个 `@Test` 只测一个场景，断言聚焦
- ✅ 鉴权 Token 在基类初始化一次，复用
- ✅ 测试数据用工厂生成，**含时间戳**避免冲突
- ✅ 测试完清理副作用（数据 / 缓存 / MQ 消息）
- ✅ 异步业务用 Awaitility 显式等待，不要 `Thread.sleep` 硬等

### 避免

- ❌ 测试间互相依赖（用例 A 必须先跑用例 B）
- ❌ 在测试里写复杂业务逻辑（应该用 Helper）
- ❌ 用真实生产数据做 setup
- ❌ 断言泛泛（`statusCode(200)` 就完了，没看 code/data）
- ❌ 在测试代码里硬编码环境 URL / 账号 / 密码

---

## 输出节奏（AI 自动生成时）

当用户说"为 xx 用例生成自动化"时，AI 应该这样输出：

1. **快速识别用例覆盖范围**（看 `test-case-generator` 的输出 或 用户提供的列表）
2. **逐个生成 `@Test` 方法**，每个方法对应 1 个用例
3. **生成或复用 Helper / DataFactory**
4. **生成或复用 ApiTestBase**
5. **生成对应 pom.xml 依赖配置**（如果是新项目）
6. **给出运行命令**（`mvn test -Dtest.env=dev`）

---

## 引用

- 上游设计：`test-case-generator` Skill
- 业务上下文：`hytech-cpa` Skill
- 代码理解：`code-walkthrough` Skill
- 协议测试：`lark-cli` Skill（结果同步到飞书）
- 《RestAssured 官方文档》https://rest-assured.io
- 《JUnit 5 用户指南》https://junit.org/junit5/docs/current/user-guide
