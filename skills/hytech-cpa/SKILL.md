# HyTech CPA 综合管理平台

## 项目概述

HyTech CPA 是一个面向注册会计师事务所的综合管理平台，包含客户管理、业务流程、审计底稿、报表生成等核心模块。

## 技术栈

- **后端**: Java 17 + Spring Boot 3.2 + Spring Security + MyBatis-Plus
- **前端**: Vue 3 + Element Plus + Vite
- **数据库**: MySQL 8.0 + Redis
- **部署**: Docker + Kubernetes

## 项目结构

```
hytech-cpa/
├── cpa-gateway/          # API 网关
├── cpa-auth/             # 认证授权服务
├── cpa-system/           # 系统管理
├── cpa-business/         # 业务核心模块
│   ├── client/           # 客户管理
│   ├── audit/            # 审计管理
│   ├── tax/              # 税务服务
│   └── report/           # 报表中心
├── cpa-common/           # 公共模块
└── cpa-ui/               # 前端项目
```

## 开发规范

### 命名规范
- Controller: `XxxController.java`
- Service 接口: `IXxxService.java`
- Service 实现: `XxxServiceImpl.java`
- Mapper: `XxxMapper.java`
- 实体类: `XxxEntity.java`
- DTO: `XxxDTO.java`

### 分支管理
- `main` — 生产分支
- `develop` — 开发分支
- `feature/xxx` — 功能分支
- `hotfix/xxx` — 紧急修复

### API 设计
- RESTful 风格
- 统一响应体: `Result<T>`
- 分页统一使用 `PageResult<T>`

## 数据库设计

### 核心表
- `sys_user` — 用户表
- `sys_role` — 角色表
- `biz_client` — 客户信息
- `biz_project` — 项目信息
- `biz_audit_paper` — 审计底稿
- `biz_tax_declaration` — 纳税申报

## 常见问题

### Q: 如何添加新的业务模块？
1. 在 `cpa-business` 下创建子模块
2. 添加 MyBatis-Plus 代码生成配置
3. 在网关中配置路由规则
4. 前端添加对应的菜单和页面

### Q: 审计底稿的权限控制逻辑？
审计底稿采用三级权限：项目经理 > 审计主管 > 审计员。底稿状态流转为：编制中 → 一级复核 → 二级复核 → 归档。
