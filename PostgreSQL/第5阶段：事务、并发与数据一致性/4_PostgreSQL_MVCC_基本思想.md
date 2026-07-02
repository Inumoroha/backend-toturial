# 4. PostgreSQL MVCC 基本思想

本节目标：理解 PostgreSQL 的 MVCC 基本直觉，知道为什么读写可以并发、为什么未提交数据不可见、为什么长事务会带来风险。

MVCC 是 Multi-Version Concurrency Control，多版本并发控制。

先不要被名字吓到。它的核心直觉是：

```text
数据被修改时，PostgreSQL 不是简单地在原地覆盖旧数据。
它会保留旧版本，同时产生新版本。
不同事务根据自己的可见性规则，看到不同版本。
```

---

## 一、为什么需要 MVCC

如果数据库只允许一个事务操作数据，其他事务全部等待，那实现起来简单，但性能很差。

真实业务中，大量请求会同时发生：

- 有人在查询商品详情。
- 有人在下单扣库存。
- 有人在查看订单列表。
- 有人在修改用户资料。

PostgreSQL 希望做到：

```text
读操作尽量不阻塞写操作。
写操作尽量不阻塞读操作。
```

MVCC 就是实现这个目标的重要机制。

---

## 二、没有 MVCC 的粗糙想象

假设一行数据：

```text
Alice balance = 1000
```

事务 A 正在把余额改成 800，但还没提交。

如果没有多版本机制，数据库可能只有一个位置存余额：

```text
balance = 800
```

这时事务 B 来查询，就会遇到问题：

- 如果看到 800，就读到了未提交数据。
- 如果不让读，就必须等待事务 A 结束。

MVCC 的思路是：

```text
旧版本 1000 继续存在。
新版本 800 也存在，但只有满足可见性规则的事务能看到。
```

这样事务 B 可以继续读取旧版本，不必等待事务 A。

---

## 三、UPDATE 不是简单覆盖

在 PostgreSQL 中，`UPDATE` 可以粗略理解为：

```text
让旧行版本失效。
插入一个新行版本。
```

例如：

```sql
UPDATE demo.accounts
SET balance = 800
WHERE owner_name = 'Alice';
```

直觉上会形成：

```text
旧版本：Alice balance = 1000
新版本：Alice balance = 800
```

事务是否能看到新版本，取决于：

- 修改它的事务是否已经提交。
- 当前事务的隔离级别。
- 当前查询使用的是哪个快照。

---

## 四、快照是什么

快照可以理解为：

```text
某一刻数据库中哪些事务已经提交、哪些事务还没提交的视图。
```

查询数据时，PostgreSQL 会根据快照判断每个行版本是否可见。

例如：

```text
事务 A 修改了 Alice 的余额，但还没提交。
事务 B 查询 Alice。
```

事务 B 的快照会认为事务 A 还没有提交，所以事务 B 看不到事务 A 产生的新版本，只能看到旧版本。

这就是为什么 PostgreSQL 不会脏读。

---

## 五、READ COMMITTED 下的快照

PostgreSQL 默认隔离级别是 `READ COMMITTED`。

在这个隔离级别下：

```text
每一条 SQL 语句开始时，都会获取一个新的快照。
```

所以同一个事务中，两次查询可能看到不同结果。

会话 A：

```sql
BEGIN;

SELECT balance
FROM demo.accounts
WHERE owner_name = 'Alice';
```

会话 B：

```sql
UPDATE demo.accounts
SET balance = balance + 100
WHERE owner_name = 'Alice';
```

会话 A 再查：

```sql
SELECT balance
FROM demo.accounts
WHERE owner_name = 'Alice';

COMMIT;
```

第二次查询会使用新的快照，因此能看到会话 B 已提交的修改。

---

## 六、REPEATABLE READ 下的快照

在 `REPEATABLE READ` 下：

```text
一个事务通常使用事务开始后的同一个快照。
```

所以同一个事务中，多次查询会看到一致结果。

会话 A：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT balance
FROM demo.accounts
WHERE owner_name = 'Alice';
```

会话 B：

```sql
UPDATE demo.accounts
SET balance = balance + 100
WHERE owner_name = 'Alice';
```

会话 A 再查：

```sql
SELECT balance
FROM demo.accounts
WHERE owner_name = 'Alice';

COMMIT;
```

会话 A 仍然看到事务快照中的旧结果。

这就是可重复读的直觉。

---

## 七、MVCC 不等于不需要锁

MVCC 让读写可以更好地并发，但它不能替代所有锁。

例如两个事务同时更新同一行：

```sql
UPDATE demo.accounts
SET balance = balance - 100
WHERE owner_name = 'Alice';
```

PostgreSQL 仍然需要对被更新的行加锁，避免两个写操作同时把同一行改乱。

可以这样理解：

```text
MVCC 主要解决读和写之间的并发可见性。
锁主要解决写和写之间的冲突，以及显式保护关键数据。
```

---

## 八、旧版本什么时候清理

因为 MVCC 会保留旧版本，所以数据库不能无限保留历史数据。

PostgreSQL 会通过 `VACUUM` 清理不再需要的旧行版本。

不再需要通常表示：

```text
没有任何还活着的事务需要看到这些旧版本。
```

这就引出了长事务问题。

如果一个事务打开很久不结束，它可能一直持有很老的快照。PostgreSQL 为了保证这个事务还能看到一致结果，就不能清理某些旧版本。

久而久之可能导致：

- 表膨胀。
- 索引膨胀。
- 查询变慢。
- `VACUUM` 清理效果变差。

---

## 九、用系统列观察版本直觉

PostgreSQL 每行都有一些系统列，例如 `xmin` 和 `xmax`。

学习阶段可以用它们辅助理解 MVCC：

```sql
SELECT xmin, xmax, id, owner_name, balance
FROM demo.accounts
ORDER BY id;
```

粗略理解：

- `xmin`：创建这个行版本的事务 ID。
- `xmax`：删除或更新这个行版本的事务 ID，未删除时通常为空或特殊值。

不要在业务代码中依赖这些系统列。这里只是帮助你建立多版本直觉。

---

## 十、MVCC 的几个重要结论

### 1. 普通 SELECT 不会读到未提交数据

这是 PostgreSQL 避免脏读的重要原因。

### 2. 读通常不会阻塞写，写通常不会阻塞读

普通 `SELECT` 可以读取旧版本，而 `UPDATE` 可以生成新版本。

### 3. 写写冲突仍然需要锁

两个事务同时修改同一行时，后来的事务通常要等待前一个事务结束。

### 4. 长事务会拖住旧版本清理

事务长时间不提交或不回滚，会让 PostgreSQL 保留更多旧版本。

---

## 十一、常见误区

### 1. MVCC 是不是让数据库完全无锁？

不是。

MVCC 减少了读写之间的阻塞，但写写冲突、DDL、显式锁等场景仍然需要锁。

### 2. UPDATE 后旧数据马上消失吗？

不是。

旧版本会保留一段时间，直到 PostgreSQL 判断没有事务需要它，再由 `VACUUM` 清理。

### 3. 快照是不是整库复制一份？

不是。

快照不是把所有数据复制一遍，而是记录事务可见性信息。查询时根据这些信息判断行版本是否可见。

### 4. 业务代码需要直接操作 MVCC 吗？

通常不需要。

你需要理解它带来的行为，例如不可重复读、可重复读、长事务风险、锁等待，而不是直接操作 MVCC。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 解释 MVCC 是多版本并发控制。
- 知道 `UPDATE` 可以理解为产生新版本，而不是简单覆盖旧版本。
- 知道 PostgreSQL 通过快照决定事务能看到哪些行版本。
- 解释为什么 PostgreSQL 不会脏读。
- 知道 `READ COMMITTED` 通常每条语句一个快照。
- 知道 `REPEATABLE READ` 通常一个事务使用同一个快照。
- 知道 MVCC 不等于没有锁。
- 知道长事务会影响旧版本清理。
