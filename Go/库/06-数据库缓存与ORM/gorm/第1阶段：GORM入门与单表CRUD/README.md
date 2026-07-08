# 01 GORM 入门

本阶段目标：从零连接数据库，定义第一个模型，并完成最基本的增删改查。

## 学习顺序

1. [安装 GORM 与连接数据库](./01-install-and-connect.md)
2. [单表 CRUD 入门](./02-basic-crud.md)
3. [错误处理与 SQL 日志](./03-error-and-debug.md)
4. [第一个完整 GORM 程序](./04-first-complete-program.md)
5. [Create 与批量插入](./05-Create与批量插入.md)
6. [First、Take、Find 查询差异](./06-First-Take-Find查询差异.md)
7. [Update、Updates、Save 与零值问题](./07-Update零值与Save差异.md)
8. [Delete、软删除与物理删除](./08-Delete软删除与物理删除.md)
9. [单表 CRUD 综合练习](./09-单表CRUD综合练习.md)

## 本阶段重点

- GORM 如何连接数据库。
- `AutoMigrate` 做了什么。
- `Create`、`First`、`Find`、`Updates`、`Delete` 的基本用法。
- 如何处理 `record not found`。
- 如何查看 GORM 生成的 SQL。

## 本阶段最终产出

你要写出一个最小可运行程序，完成 `User` 表的增删改查。
