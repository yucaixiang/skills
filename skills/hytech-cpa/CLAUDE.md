# hytech-cpa 项目编码准则

在 hytech-cpa 项目中编写、修改、重构、审查任何代码之前，必须先调用 `hytech-cpa` skill 加载项目知识，然后严格遵守以下硬规则。

## 必读知识

写代码前，根据要做的事情先读：

| 改动类型 | 必读文档 |
|---|---|
| 任意改动 | `hytech-cpa/SKILL.md` + `hytech-cpa/docs/05-development-guide.md` |
| 改 Controller/Service | + `hytech-cpa/docs/02-business-modules.md` |
| 改 Entity/Mapper/SQL | + `hytech-cpa/docs/03-data-model.md` |
| 排查佣金/分润/结算异常 | + `hytech-cpa/docs/06-business-flows.md` + `07-faq.md` |
| 配置 Redis/Kafka | + `hytech-cpa/docs/04-tech-stack.md` |

## 6 条硬规则（违反即驳回）

### 规则 1：分层架构不可逾越
- `application` 层只做协议转换与技术适配，**禁止包含业务规则**
- `business` 层做服务编排，**禁止直接访问其他业务的数据库**
- `backend` 层只做本业务数据原子访问，**禁止对外返回 DB 实体（Entity / PO），必须用 DTO**
- `api` 层只做接口契约封装，**禁止包含数据验证规则**

### 规则 2：领域间禁止互调
- ToB（admin）、ToC（client）、ToJ（job）、ToO（open）**不能直接调用对方的 Controller/Service**
- 跨域必须通过 `*-platform-api` Feign 接口 + 数据契约 DTO
- 同域内的服务间通过 `*-internal-api` 调用

### 规则 3：接口前缀强约束
- `backend` 层接口 URL 必须以 `/bck/` 开头
- `business` 层接口 URL 必须以 `/bsn/` 开头
- 例：`/bsn/cpa-account/page`、`/bck/cpa-account/query-by-id`

### 规则 4：包结构必须遵守
```
controller       控制层
service          服务接口
service/impl     服务实现
model/req        API 入参，类名以 Req 结尾
model/res        API 出参，类名以 Res 结尾
model/db         DB 实体类，类名为表名驼峰
model/dto        数据传输类，类名以 DTO 结尾
mapper           MyBatis Mapper
enums            枚举类
constant         静态常量
config           配置类
jobs             XxlJob 任务类，以 Job 结尾
util             工具类
```

### 规则 5：Redis Key 命名规范
- 格式：`域:业务:模块:功能:数据`
- 全小写，冒号分隔
- 示例：`partner:cpa:leads:info:{leadsId}`、`partner:cpa:leads:allocate:lock:{leadsId}`

### 规则 6：Kafka Topic 命名规范
- 格式：`域.业务.模块.功能`
- 全小写，点号分隔
- 示例：`partner.cpa.leads.allocate`、`partner.cpa.bonus.calculate`

## 通用编码原则

### 必须做
- 使用 Lombok（`@Getter` `@Setter` `@Builder` `@Slf4j` 等）
- 使用 MapStruct 做 Entity ↔ DTO 转换
- 使用 `@Resource` 注入（项目惯例，少数地方使用 `@Autowired`）
- 关键流程加 `log.info` 携带 `clientUserId` / `cpaAccount` 等业务标识，便于排查
- 涉及并发的写操作使用 `RedissonLock` 加分布式锁
- 异常用 `BizException` + `JobResponseEnum` / `AdminResponseEnum` 等枚举

### 必须避免
- 在 controller 里写业务逻辑（必须下沉到 service）
- 在 backend 里集成外部 API（外部 API 调用必须在 business 层）
- 用 Entity 作为接口入参/出参
- 在 business 层重复造 backend 已有的查询方法
- 用裸字符串拼 Redis key / Kafka topic（必须用常量）

## 改动前的标准动作

1. 调用 `hytech-cpa` skill 了解项目（如果尚未加载）
2. 确认要改的代码在哪一层（admin/client/job/open · business/backend · service/controller）
3. 检查该层是否允许做这件事（参考规则 1-2）
4. 检查类名 / 接口路径是否符合规范（参考规则 3-4）
5. 写代码 → 自检 → 自测
6. 提交时 commit message 带上业务上下文（如 "feat(bonus): xxx"）

---

> 备注：如果发现项目里已有代码违反了上面的规则，**不要顺手改**（外科手术原则）— 在 commit message 或 review 中标注，让 owner 决定是否单独修复。
