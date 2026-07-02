# 2. ACID：事务的四个特性

本节目标：理解 ACID 的含义，知道原子性、一致性、隔离性、持久性分别解决什么问题，并能把它们和真实业务联系起来。

ACID 是事务最经典的四个特性：

```text
A: Atomicity，原子性
C: Consistency，一致性
I: Isolation，隔离性
D: Durability，持久性
```

这四个词看起来抽象，但它们背后都是非常具体的业务问题。

---

## 一、Atomicity：原子性

原子性表示：

```text
一个事务中的多个操作，要么全部成功，要么全部失败。
```

还是转账场景：

```text
Alice 扣 100 元。
Bob 加 100 元。
记录转账流水。
```

这些操作必须作为一个整体。

如果任何一步失败，都应该回滚整个事务。

```sql
BEGIN;

UPDATE demo.accounts
SET balance = balance - 100
WHERE owner_name = 'Alice';

UPDATE demo.accounts
SET balance = balance + 100
WHERE owner_name = 'Bob';

INSERT INTO demo.account_logs (account_id, amount, log_type, remark)
SELECT id, -100, 'transfer_out', 'transfer to Bob'
FROM demo.accounts
WHERE owner_name = 'Alice';

INSERT INTO demo.account_logs (account_id, amount, log_type, remark)
SELECT id, 100, 'transfer_in', 'transfer from Alice'
FROM demo.accounts
WHERE owner_name = 'Bob';

COMMIT;
```

如果中途失败：

```sql
ROLLBACK;
```

原子性关注的是：

```text
事务内部的操作能不能作为不可拆分的整体。
```

---

## 二、Consistency：一致性

一致性表示：

```text
事务执行前后，数据都必须满足数据库约束和业务规则。
```

数据库约束包括：

- `PRIMARY KEY`
- `UNIQUE`
- `NOT NULL`
- `CHECK`
- `FOREIGN KEY`

例如账户余额不能为负数：

```sql
CONSTRAINT accounts_balance_check CHECK (balance >= 0)
```

如果执行：

```sql
UPDATE demo.accounts
SET balance = -1
WHERE owner_name = 'Alice';
```

PostgreSQL 会拒绝这次修改。

这就是数据库约束帮助维护一致性。

不过要注意：一致性不完全由数据库自动保证。业务规则也需要你自己正确建模和写代码。

例如：

```text
订单 total_amount 必须等于 product.price * quantity。
```

数据库可以通过字段类型、约束、外键帮你守住一部分规则，但复杂业务逻辑通常还需要应用代码配合。

---

## 三、Isolation：隔离性

隔离性表示：

```text
多个事务同时执行时，彼此之间不能随意看到对方未完成的中间状态。
```

假设事务 A 正在转账：

```text
Alice 已经扣 100。
Bob 还没加 100。
事务 A 尚未提交。
```

如果事务 B 此时能看到这个中间状态，就可能认为系统总余额减少了 100。

这就是隔离性要避免的问题。

隔离性不是说事务之间完全不能互相影响。不同隔离级别允许的可见范围不同。

PostgreSQL 常用隔离级别包括：

- `READ COMMITTED`
- `REPEATABLE READ`
- `SERIALIZABLE`

默认是 `READ COMMITTED`。

隔离性关注的是：

```text
并发事务之间能看到什么，不能看到什么。
```

---

## 四、Durability：持久性

持久性表示：

```text
事务一旦提交成功，数据就应该可靠保存下来。
```

也就是说，当数据库告诉你：

```sql
COMMIT;
```

成功后，即使数据库进程崩溃、服务器重启，已经提交的数据也应该能恢复。

PostgreSQL 主要通过 WAL 实现持久性。

WAL 是 Write-Ahead Logging，预写式日志。你可以先有一个直觉：

```text
真正修改数据文件之前，PostgreSQL 会先记录日志。
发生崩溃时，可以根据日志恢复已经提交的事务。
```

持久性关注的是：

```text
COMMIT 成功之后，数据会不会丢。
```

---

## 五、ACID 放到一个转账例子里

假设 Alice 给 Bob 转账 100 元。

### 原子性

Alice 扣款、Bob 加款、流水记录必须一起成功。

如果 Bob 加款失败，Alice 扣款也要撤销。

### 一致性

转账前后都要满足规则：

- 账户余额不能为负数。
- 流水必须引用真实存在的账户。
- 转账金额必须合法。

### 隔离性

转账过程中，其他事务不能看到“只扣了 Alice 但还没加 Bob”的中间状态。

### 持久性

转账提交成功后，即使数据库重启，转账结果也不应该丢失。

---

## 六、ACID 和后端代码的关系

数据库提供 ACID 能力，但后端代码必须正确使用它。

错误示例：

```text
先扣库存，自动提交。
再创建订单，自动提交。
如果创建订单失败，库存已经被扣掉。
```

正确思路：

```text
开启事务。
检查并扣减库存。
创建订单。
提交事务。
如果出错，回滚事务。
```

也就是说，数据库提供的是工具，业务代码负责划清事务边界。

---

## 七、常见误区

### 1. ACID 是数据库自动帮我处理所有业务问题吗？

不是。

数据库负责提供事务能力、约束能力、日志能力、隔离能力。业务规则是否正确，还取决于表设计和代码实现。

### 2. 一致性是不是等于数据完全正确？

不完全是。

数据库只能保证它知道的规则。比如你定义了 `CHECK (balance >= 0)`，它就能防止负余额。你没有定义或没有在代码中处理的业务规则，数据库无法凭空知道。

### 3. 隔离性越高越好吗？

不一定。

隔离性越高，越接近串行执行，能避免更多并发异常，但也可能带来更多等待、冲突和重试成本。

真实项目里要根据业务风险选择合适隔离级别和锁策略。

### 4. COMMIT 成功就绝对不会丢数据吗？

正常情况下，数据库会尽力保证已提交事务的持久性。

但生产系统的数据可靠性还依赖磁盘、文件系统、数据库配置、备份策略和运维方案。对后端开发来说，先建立“提交成功才算真正生效”的基本认知即可。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 ACID 分别代表什么。
- 用转账例子说明原子性。
- 用约束和业务规则说明一致性。
- 用并发事务说明隔离性。
- 用 `COMMIT` 和 WAL 的直觉说明持久性。
- 知道数据库提供 ACID 能力，但业务代码必须正确划分事务边界。
