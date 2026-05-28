# 测试策略

## 测试金字塔

```
     /  E2E  \        少量关键路径
    / 集成测试 \       API级别验证
   /  单元测试  \      覆盖核心逻辑
```

## 单元测试规范
- 框架: JUnit 5 + Mockito
- 覆盖率要求: 行覆盖 >= 80%
- 命名: `should_行为_when_条件`
- 每个测试只验证一个行为

## 集成测试
- 使用 TestContainers 启动依赖
- 测试 API 的完整请求/响应
- 数据库操作验证
- 外部服务 Mock (WireMock)

## E2E 测试
- 框架: Playwright
- 覆盖核心业务流程
- 在 CI 中每次部署前运行
- 失败自动截图和录屏

## 测试数据
- 单元测试: 使用 Builder 模式构造
- 集成测试: SQL 脚本初始化
- E2E: 专用测试环境，定期重置

## CI 集成
```yaml
test:
  stage: test
  script:
    - mvn test           # 单元测试
    - mvn verify         # 集成测试
    - npm run test:e2e   # E2E测试
  coverage: '/Total.*?(\d+%)/'
```
