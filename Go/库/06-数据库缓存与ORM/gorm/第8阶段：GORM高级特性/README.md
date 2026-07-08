# 08 高级特性

本阶段目标：掌握 GORM 在复杂项目中的高级能力，包括 Hook、Scope、自定义类型、锁、上下文和 GORM Gen。

## 学习顺序

1. [Hook、Scope 与 Context](./01-hook-scope-context.md)
2. [自定义类型、锁与多数据库](./02-custom-type-lock-dbresolver.md)
3. [GORM Gen 类型安全查询](./03-gorm-gen.md)
4. [高级特性综合练习](./04-advanced-practice.md)
5. [Hook 生命周期详解](./05-Hook生命周期详解.md)
6. [Scope 复用查询条件](./06-Scope复用查询条件.md)
7. [Context 超时与链路传递](./07-Context超时与链路传递.md)
8. [JSON 字段与自定义类型](./08-JSON字段与自定义类型.md)
9. [锁机制专题](./09-锁机制专题.md)
10. [Gen 与读写分离进阶路线](./10-Gen与读写分离进阶路线.md)
11. [高级特性项目化实践](./11-高级特性项目化实践.md)
12. [高级特性取舍清单](./12-高级特性取舍清单.md)

## 本阶段重点

- Hook 适合做什么，不适合做什么。
- 用 Scope 复用查询条件。
- 用 Context 控制超时和传递请求信息。
- 自定义 JSON 字段。
- 悲观锁和乐观锁。
- 多数据库和读写分离的基本认识。
- GORM Gen 的价值。
