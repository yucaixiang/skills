# 开发规范

## 代码风格

### Java
- 使用 Google Java Style
- 方法不超过 50 行
- 类不超过 500 行
- 必须有单元测试覆盖

### 前端
- ESLint + Prettier 统一格式化
- 组件文件使用 PascalCase
- 工具函数使用 camelCase

## Git Flow

### 分支命名
- `feature/JIRA-123-short-desc`
- `bugfix/JIRA-456-fix-desc`
- `hotfix/JIRA-789-urgent-fix`

### Commit 规范
使用 Conventional Commits:
- `feat: 新功能`
- `fix: 修复bug`
- `docs: 文档更新`
- `refactor: 重构`
- `test: 测试`

## Code Review 标准
1. 功能正确性
2. 代码可读性
3. 性能考虑
4. 安全检查
5. 测试覆盖率 >= 80%
