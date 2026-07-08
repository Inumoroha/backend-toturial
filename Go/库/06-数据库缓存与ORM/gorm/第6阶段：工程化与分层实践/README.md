# 06 工程化实践

本阶段目标：把 GORM 放进真实 Go 后端项目结构中，而不是只写单文件 demo。你要学会数据库初始化、配置管理、分层封装、DTO 分离和测试。

## 学习顺序

1. [项目结构与数据库初始化](./01-project-structure-database.md)
2. [Repository、Service、Handler 分层](./02-repository-service-handler.md)
3. [DTO、错误处理与测试](./03-dto-error-testing.md)
4. [博客 API 分层实现 walkthrough](./04-blog-api-implementation-walkthrough.md)
5. [配置模块设计](./05-配置模块设计.md)
6. [数据库模块设计](./06-数据库模块设计.md)
7. [Repository 接口设计](./07-Repository接口设计.md)
8. [Service 业务编排](./08-Service业务编排.md)
9. [Handler 与 DTO 边界](./09-Handler与DTO边界.md)
10. [工程化测试策略](./10-工程化测试策略.md)
11. [main 函数组装全流程](./11-main函数组装全流程.md)

## 本阶段重点

- 数据库连接只初始化一次。
- GORM 操作集中在 Repository 层。
- Service 层处理业务规则和事务。
- Handler 层只处理 HTTP 输入输出。
- 不把 GORM Model 直接当接口响应。
- 给关键 Repository 和 Service 写测试。

## 本阶段最终产出

把博客系统改造成一个 HTTP API 项目，至少包含：

- 用户注册
- 用户登录
- 创建文章
- 编辑文章
- 文章列表
- 文章详情
- 创建评论
