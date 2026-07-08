# 第 6 阶段：事务、一致性与并发

本阶段学习 MongoDB 中最容易出线上事故的一组主题：原子性、事务、一致性、并发写入、幂等、库存扣减、钱包转账。

你的目标不是“凡是复杂就上事务”，而是学会判断：

- 单文档原子更新能不能解决？
- 唯一索引和幂等键能不能解决？
- 乐观锁能不能解决？
- 什么时候确实需要多文档事务？
- 事务失败、重试、超时、并发冲突该怎么处理？

## 本阶段目标

完成本阶段后，你应该能做到：

- 理解 MongoDB 单文档原子性。
- 使用条件更新防止库存扣成负数。
- 使用唯一索引和幂等键防止重复提交。
- 理解多文档事务的适用场景和成本。
- 使用 Go Driver v2 的 `StartSession` 和 `WithTransaction`。
- 理解 readConcern、writeConcern、readPreference 的基本含义。
- 使用乐观锁处理并发更新。
- 设计钱包转账事务。
- 设计库存扣减和库存流水。
- 判断什么时候不该使用事务。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-单文档原子性与条件更新.md` | 单文档原子性和库存扣减基础 |
| 02 | `02-并发问题与唯一索引幂等.md` | 重复提交、唯一索引、幂等键 |
| 03 | `03-乐观锁与版本号.md` | 用 version/updated_at 解决并发覆盖 |
| 04 | `04-多文档事务适用场景与成本.md` | 什么时候用事务，什么时候不用 |
| 05 | `05-mongosh事务示例.md` | 在 mongosh 中理解 session/commit/abort |
| 06 | `06-Go-Driver-v2事务写法.md` | Go 中使用 `WithTransaction` |
| 07 | `07-readConcern-writeConcern-readPreference.md` | 一致性相关概念 |
| 08 | `08-钱包转账实战设计.md` | 钱包扣款、加款、流水事务 |
| 09 | `09-库存扣减与防重复下单.md` | 库存、订单、流水、幂等 |
| 10 | `10-阶段练习-交易与库存系统.md` | 综合练习 |
| 11 | `11-阶段验收与并发安全清单.md` | 验收和评审清单 |

## 本阶段数据库

统一使用：

```javascript
use transaction_stage6
```

## 一个重要判断

很多并发问题不需要事务。

例如库存扣减：

```javascript
db.products.updateOne(
  { _id: productId, stock: { $gte: quantity } },
  { $inc: { stock: -quantity } }
)
```

这个单文档条件更新已经是原子的。

但钱包转账通常涉及：

```text
A 钱包扣款
B 钱包加款
写交易流水
```

这跨多个文档，通常需要事务。

## 官方资料

本阶段主要参考：

- Atomicity and Transactions：<https://www.mongodb.com/docs/manual/core/write-operations-atomicity/>
- Transactions：<https://www.mongodb.com/docs/manual/core/transactions/>
- Read Concern：<https://www.mongodb.com/docs/manual/reference/read-concern/>
- Write Concern：<https://www.mongodb.com/docs/manual/reference/write-concern/>
- Read Preference：<https://www.mongodb.com/docs/manual/core/read-preference/>
- Go Driver Transactions：<https://www.mongodb.com/docs/drivers/go/current/fundamentals/transactions/>

