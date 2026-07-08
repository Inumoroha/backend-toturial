# PostgreSQL 面试题与参考答案

这份文档用于准备后端实习面试中的 PostgreSQL 与数据库相关问题。

它不是为了让你死记硬背，而是帮助你做到：

```text
听得懂面试官在问什么。
知道该从哪个角度回答。
能结合自己的 Go + PostgreSQL 项目解释。
遇到追问不慌。
```

建议复习方式：

1. 先看题目，自己尝试回答。
2. 再看参考答案。
3. 最后把答案改成自己的表达。
4. 尽量结合短链接项目、用户模块、订单系统这类例子讲。

---

## 一、数据库基础类

### 1. 什么是数据库？什么是关系型数据库？

参考回答：

数据库是用来持久化存储、管理和查询数据的系统。

关系型数据库使用表来组织数据，表由行和列组成。每一行表示一条记录，每一列表示一个字段。不同表之间可以通过主键、外键等方式表达关系。

例如用户系统中：

```text
users 表保存用户。
posts 表保存文章。
posts.user_id 关联 users.id。
```

PostgreSQL、MySQL 都是关系型数据库。

追问点：

- 关系型数据库和 MongoDB 有什么区别？
- 为什么后端项目常用关系型数据库？

可以补充：

关系型数据库擅长结构清晰、关系明确、需要事务和约束的数据场景。比如用户、订单、支付、库存等业务，通常更适合关系型数据库。

---

### 2. PostgreSQL 和 MySQL 有什么区别？

参考回答：

PostgreSQL 和 MySQL 都是常见的关系型数据库。

PostgreSQL 更强调标准 SQL、复杂查询能力、扩展能力和数据一致性。它支持很多强大的类型和能力，例如 `jsonb`、数组、范围类型、GIN 索引、全文搜索、PostGIS 扩展等。

MySQL 使用也非常广泛，生态成熟，在很多 Web 项目里很常见。

如果是新项目，并且希望有更强的 SQL 能力、复杂查询能力、JSONB、扩展能力，PostgreSQL 是很好的选择。

面试中不要贬低 MySQL，可以这样说：

```text
两者都很成熟。我的学习和项目主要使用 PostgreSQL，所以更熟悉 PostgreSQL 的类型系统、事务、索引和 pgx 接入方式。
```

---

### 3. 什么是 schema？

参考回答：

schema 可以理解为数据库中的命名空间，用来组织表、视图、函数等对象。

PostgreSQL 默认有 `public` schema，但项目中可以创建自己的 schema，例如：

```sql
CREATE SCHEMA app;
```

然后创建表：

```sql
CREATE TABLE app.users (
    id bigserial PRIMARY KEY,
    email text NOT NULL
);
```

使用 schema 的好处是可以让数据库对象组织更清晰，也方便权限管理。

---

### 4. char、varchar、text 有什么区别？

参考回答：

`char(n)` 是固定长度字符串，不足会补空格。

`varchar(n)` 是可变长度字符串，但限制最大长度。

`text` 是可变长度字符串，不需要指定长度。

在 PostgreSQL 中，`text` 和 `varchar` 性能上通常没有明显差异。实际项目里，如果业务没有明确长度限制，我通常会使用 `text`。如果业务上确实有长度规则，例如用户名最多 30 个字符，可以用 `CHECK` 约束或者在应用层校验。

示例：

```sql
CREATE TABLE app.users (
    email text NOT NULL,
    username text NOT NULL,
    CONSTRAINT username_length_check CHECK (length(username) <= 30)
);
```

---

### 5. timestamp 和 timestamptz 有什么区别？

参考回答：

`timestamp` 不带时区语义，表示一个普通的日期时间。

`timestamptz` 是带时区语义的时间戳。PostgreSQL 会按时区处理输入和输出，内部保存的是统一时间点。

后端项目里更推荐使用 `timestamptz` 保存创建时间、更新时间、支付时间、登录时间等事件发生时间。

例如：

```sql
created_at timestamptz NOT NULL DEFAULT now()
```

这样在跨时区系统里更稳。

---

## 二、SQL 基础与 CRUD

### 6. SQL 分为哪几类？

参考回答：

常见可以分为：

- DDL：数据定义语言，例如 `CREATE TABLE`、`ALTER TABLE`、`DROP TABLE`。
- DML：数据操作语言，例如 `INSERT`、`UPDATE`、`DELETE`。
- DQL：数据查询语言，主要是 `SELECT`。
- DCL：权限控制，例如 `GRANT`、`REVOKE`。
- TCL：事务控制，例如 `BEGIN`、`COMMIT`、`ROLLBACK`。

实际开发中最常写的是 `SELECT`、`INSERT`、`UPDATE`、`DELETE`，表结构变更则通过迁移工具管理。

---

### 7. DELETE、TRUNCATE、DROP 有什么区别？

参考回答：

`DELETE` 删除表中的行，可以带 `WHERE` 条件，属于 DML。

```sql
DELETE FROM users WHERE id = 1;
```

`TRUNCATE` 清空整张表，通常比 `DELETE` 全表更快，但限制更多，会直接清空表数据。

```sql
TRUNCATE TABLE users;
```

`DROP` 是删除数据库对象，例如删除整张表，表结构也没了。

```sql
DROP TABLE users;
```

生产环境中这三个都要非常谨慎，尤其是 `TRUNCATE` 和 `DROP`。

---

### 8. UPDATE 忘记 WHERE 会发生什么？

参考回答：

会更新整张表。

例如：

```sql
UPDATE users
SET status = 'disabled';
```

这会把所有用户状态都改成 `disabled`。

所以生产环境执行 `UPDATE` 或 `DELETE` 时一定要确认 `WHERE` 条件，必要时先用同样条件执行 `SELECT count(*)` 看影响行数。

```sql
SELECT count(*)
FROM users
WHERE status = 'inactive';
```

---

### 9. 什么是 INSERT ... RETURNING？

参考回答：

PostgreSQL 支持 `RETURNING`，可以在插入、更新、删除后直接返回受影响的数据。

例如创建用户：

```sql
INSERT INTO app.users (email, username)
VALUES ($1, $2)
RETURNING id, email, username, created_at;
```

这样 Go 代码不用再额外查一次数据库，就能拿到数据库生成的 `id` 和 `created_at`。

在短链接项目中，创建短链时可以：

```sql
INSERT INTO app.short_links (code, original_url)
VALUES ($1, $2)
RETURNING id, code, original_url, visit_count, created_at, updated_at;
```

---

### 10. 什么是 UPSERT？

参考回答：

UPSERT 指的是“如果不存在就插入，如果已存在就更新”。

PostgreSQL 使用：

```sql
INSERT ... ON CONFLICT ... DO UPDATE
```

例如：

```sql
INSERT INTO app.users (email, username)
VALUES ($1, $2)
ON CONFLICT (email)
DO UPDATE SET username = EXCLUDED.username
RETURNING id, email, username;
```

`EXCLUDED` 表示本次尝试插入但发生冲突的那行数据。

常见场景：

- 用户邮箱已存在时更新用户信息。
- 统计表累加计数。
- 幂等写入。

---

## 三、表设计与约束

### 11. 主键是什么？为什么需要主键？

参考回答：

主键用于唯一标识表中的一行记录。

主键必须唯一，且不能为 `NULL`。

例如：

```sql
CREATE TABLE app.users (
    id bigserial PRIMARY KEY,
    email text NOT NULL
);
```

主键的作用：

- 唯一定位一条记录。
- 被其他表作为外键引用。
- 帮助数据库建立索引。
- 让数据模型更稳定。

实际项目中，常用主键类型有自增整数、bigint、uuid。

---

### 12. 自增 ID 和 UUID 怎么选择？

参考回答：

自增 ID 简单、高效、索引局部性好，适合作为内部主键。

UUID 不容易被猜到，适合作为公开资源 ID，也适合分布式生成。

但 UUID 比整数更大，索引占用更多空间。

我的理解是：

```text
内部简单业务可以用 bigint 自增主键。
对外暴露的资源 ID 可以用 uuid。
```

例如短链接项目中：

```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid()
code text NOT NULL UNIQUE
```

`id` 是资源主键，`code` 是业务唯一短码。

---

### 13. 什么是外键？一定要使用外键吗？

参考回答：

外键用于保证两张表之间的引用关系。

例如文章表引用用户表：

```sql
CREATE TABLE app.posts (
    id bigserial PRIMARY KEY,
    user_id bigint NOT NULL REFERENCES app.users(id),
    title text NOT NULL
);
```

这样数据库会保证 `posts.user_id` 对应的用户必须存在。

外键的优点：

- 保证数据一致性。
- 防止产生无效引用。
- 表达业务关系清楚。

是否一定使用，要看项目情况。有些高并发大规模系统会出于性能、发布、分库等考虑减少数据库外键，改由应用层保证。但对大多数普通项目和学习项目，合理使用外键有助于保证数据正确。

---

### 14. UNIQUE 约束和普通索引有什么区别？

参考回答：

普通索引用于加速查询，不保证数据唯一。

`UNIQUE` 约束既会创建唯一索引，又会保证数据不能重复。

例如：

```sql
CREATE TABLE app.users (
    id bigserial PRIMARY KEY,
    email text NOT NULL UNIQUE
);
```

它不仅能加速：

```sql
SELECT * FROM app.users WHERE email = $1;
```

还可以防止插入重复邮箱。

业务唯一性必须靠数据库约束兜底，不能只靠应用层先查再插入。

---

### 15. NOT NULL 和 CHECK 有什么用？

参考回答：

`NOT NULL` 保证字段不能为空。

```sql
email text NOT NULL
```

`CHECK` 用于定义更具体的规则。

例如：

```sql
age integer CHECK (age >= 0)
```

短链接项目中可以写：

```sql
CONSTRAINT short_links_visit_count_non_negative_check CHECK (visit_count >= 0)
```

约束的意义是把重要的数据规则放到数据库层保护，避免应用 bug 写入非法数据。

---

### 16. 软删除怎么设计？

参考回答：

软删除通常不是直接 `DELETE` 数据，而是加字段标记删除状态。

常见字段：

```sql
deleted_at timestamptz
```

查询时过滤：

```sql
WHERE deleted_at IS NULL
```

优点：

- 可以恢复。
- 可以保留历史。
- 避免误删。

缺点：

- 查询都要注意过滤。
- 唯一约束可能更复杂。
- 表数据会越来越多。

如果软删除后邮箱允许重新注册，可以使用部分唯一索引：

```sql
CREATE UNIQUE INDEX users_email_active_unique
ON app.users (email)
WHERE deleted_at IS NULL;
```

---

## 四、查询、JOIN 与聚合

### 17. INNER JOIN 和 LEFT JOIN 有什么区别？

参考回答：

`INNER JOIN` 只返回两边都匹配的记录。

`LEFT JOIN` 返回左表所有记录，即使右表没有匹配，右表字段也会是 `NULL`。

例如查询文章和作者：

```sql
SELECT p.id, p.title, u.username
FROM app.posts p
INNER JOIN app.users u ON u.id = p.user_id;
```

如果要查询所有用户以及他们的文章数量，即使用户没有文章也要返回，就可以用 `LEFT JOIN`。

```sql
SELECT u.id, u.username, count(p.id) AS post_count
FROM app.users u
LEFT JOIN app.posts p ON p.user_id = u.id
GROUP BY u.id, u.username;
```

---

### 18. WHERE 和 HAVING 有什么区别？

参考回答：

`WHERE` 在分组前过滤行。

`HAVING` 在 `GROUP BY` 聚合后过滤分组结果。

例如查询年龄大于 18 的用户：

```sql
SELECT *
FROM users
WHERE age > 18;
```

查询文章数量大于 10 的用户：

```sql
SELECT user_id, count(*) AS post_count
FROM posts
GROUP BY user_id
HAVING count(*) > 10;
```

简单说：

```text
WHERE 过滤原始行。
HAVING 过滤聚合后的组。
```

---

### 19. count(*)、count(1)、count(column) 有什么区别？

参考回答：

`count(*)` 统计行数。

`count(1)` 通常也统计行数，在 PostgreSQL 中和 `count(*)` 没必要刻意区分。

`count(column)` 只统计该列不为 `NULL` 的行数。

例如：

```sql
SELECT count(*) FROM users;
SELECT count(email) FROM users;
```

如果 `email` 有 `NULL`，`count(email)` 会少于 `count(*)`。

实际项目统计行数时，优先写 `count(*)`，语义清楚。

---

### 20. LIMIT/OFFSET 分页有什么问题？

参考回答：

`LIMIT/OFFSET` 简单直观。

例如：

```sql
SELECT *
FROM app.short_links
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;
```

问题是 offset 很大时，数据库仍然要扫描并跳过前面的很多行，再返回后面的 20 行。

数据量大时会变慢。

更好的方式是游标分页，例如基于 `created_at` 和 `id`：

```sql
SELECT *
FROM app.short_links
WHERE (created_at, id) < ($1, $2)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

实习面试中可以说：

```text
小数据量或后台简单列表可以先用 LIMIT/OFFSET，大数据量和深分页更推荐游标分页。
```

---

## 五、事务与 ACID

### 21. 什么是事务？

参考回答：

事务是一组数据库操作的执行单元，要么全部成功，要么全部失败。

典型例子是转账：

```text
A 账户扣钱。
B 账户加钱。
```

这两个操作必须一起成功或一起失败。

SQL 示例：

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

如果中间出错：

```sql
ROLLBACK;
```

---

### 22. ACID 是什么？

参考回答：

ACID 是事务的四个特性：

`A` Atomicity，原子性：事务中的操作要么全部成功，要么全部失败。

`C` Consistency，一致性：事务执行前后，数据要满足约束和业务规则。

`I` Isolation，隔离性：并发事务之间互相隔离，不应该随意看到彼此的中间状态。

`D` Durability，持久性：事务提交后，数据应该持久保存，即使系统故障也不应轻易丢失。

项目表达：

```text
例如订单创建和库存扣减必须放在事务里，避免订单创建成功但库存没扣，或者库存扣了但订单失败。
```

---

### 23. COMMIT 和 ROLLBACK 分别做什么？

参考回答：

`COMMIT` 提交事务，让事务中的修改永久生效。

`ROLLBACK` 回滚事务，撤销事务中尚未提交的修改。

例如：

```sql
BEGIN;
UPDATE users SET name = 'Alice' WHERE id = 1;
ROLLBACK;
```

这里更新不会生效。

如果是：

```sql
BEGIN;
UPDATE users SET name = 'Alice' WHERE id = 1;
COMMIT;
```

更新会生效。

---

### 24. Go 中使用事务要注意什么？

参考回答：

使用事务时要保证任何路径都能提交或回滚。

常见写法：

```go
tx, err := pool.Begin(ctx)
if err != nil {
    return err
}
defer tx.Rollback(ctx)

// 执行多条 SQL

if err := tx.Commit(ctx); err != nil {
    return err
}
```

`defer tx.Rollback(ctx)` 可以保证中间出错时事务不会一直挂着。

事务里不要做耗时的外部操作，比如调用第三方接口、等待用户输入、处理大文件。事务边界要尽量小。

---

## 六、并发问题与隔离级别

### 25. 什么是脏读、不可重复读、幻读？

参考回答：

脏读：一个事务读到了另一个事务还没有提交的数据。如果对方回滚，读到的数据就是脏的。

不可重复读：同一个事务中，两次读取同一行，结果不一样，因为中间有其他事务提交了修改。

幻读：同一个事务中，两次按条件查询，第二次出现了第一次没有的行，通常是其他事务插入了新数据。

PostgreSQL 默认隔离级别是 `READ COMMITTED`，不会出现脏读。

---

### 26. PostgreSQL 有哪些事务隔离级别？

参考回答：

PostgreSQL 支持：

- `READ COMMITTED`
- `REPEATABLE READ`
- `SERIALIZABLE`

另外 SQL 标准里有 `READ UNCOMMITTED`，但 PostgreSQL 中实际按 `READ COMMITTED` 处理，不允许脏读。

默认是：

```sql
READ COMMITTED
```

大多数普通后端业务默认隔离级别已经够用。只有在需要更强一致性时，才考虑更高隔离级别。

---

### 27. READ COMMITTED 是什么意思？

参考回答：

`READ COMMITTED` 表示一条 SQL 只能看到已经提交的数据。

在同一个事务里，不同 SQL 执行时可能看到不同的已提交数据。

所以它可以避免脏读，但可能出现不可重复读。

PostgreSQL 默认使用这个隔离级别，因为它在一致性和并发性能之间比较平衡。

---

### 28. 什么是 MVCC？

参考回答：

MVCC 是多版本并发控制。

PostgreSQL 通过保存数据的多个版本，让读写可以更好地并发执行。

简单理解：

```text
写操作不会简单覆盖旧数据，而是产生新版本。
不同事务根据自己的快照看到不同版本。
```

这样读操作通常不需要阻塞写操作，写操作也不一定阻塞读操作。

但 MVCC 会产生旧版本数据，所以 PostgreSQL 需要 VACUUM 清理不再需要的旧版本。

---

### 29. 什么是行锁？什么时候会出现？

参考回答：

行锁是数据库对某一行数据加的锁，常见于 `UPDATE`、`DELETE` 或 `SELECT ... FOR UPDATE`。

例如：

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 1;
```

这会锁住被更新的商品行。

如果另一个事务同时更新同一行，需要等待前一个事务结束。

行锁可以保证并发更新同一行时数据不会互相覆盖。

---

### 30. 什么是死锁？

参考回答：

死锁是两个或多个事务互相等待对方释放锁，导致谁也无法继续。

例如：

```text
事务 A 锁住用户 1，想锁用户 2。
事务 B 锁住用户 2，想锁用户 1。
```

双方互相等待，就形成死锁。

PostgreSQL 会检测死锁，并中止其中一个事务。

避免死锁的方法：

- 多个事务按相同顺序访问资源。
- 事务尽量短。
- 避免用户交互放在事务里。
- 捕获死锁错误并重试。

---

## 七、索引与性能

### 31. 索引是什么？为什么能加快查询？

参考回答：

索引是数据库为了加速查询而维护的数据结构。

可以类比书的目录。

如果没有索引，数据库可能需要扫描整张表。使用索引后，数据库可以更快定位到符合条件的数据。

例如：

```sql
CREATE INDEX users_email_idx ON app.users (email);
```

查询：

```sql
SELECT *
FROM app.users
WHERE email = $1;
```

就可能使用索引。

但索引不是免费的，它会增加写入成本和存储成本。

---

### 32. B-Tree 索引适合哪些查询？

参考回答：

PostgreSQL 默认索引类型是 B-Tree。

B-Tree 适合：

- 等值查询：`=`
- 范围查询：`>`、`<`、`BETWEEN`
- 排序：`ORDER BY`
- 前缀匹配的某些场景

例如：

```sql
CREATE INDEX orders_created_at_idx ON orders (created_at);
```

适合：

```sql
WHERE created_at >= $1 AND created_at < $2
ORDER BY created_at DESC
```

---

### 33. 组合索引的顺序为什么重要？

参考回答：

组合索引是多个列组成的索引。

例如：

```sql
CREATE INDEX orders_user_created_idx
ON orders (user_id, created_at DESC);
```

它适合：

```sql
WHERE user_id = $1
ORDER BY created_at DESC
```

索引顺序很重要，因为数据库会按照索引列的顺序组织数据。

一般把高频过滤条件、等值条件放在前面，再考虑排序字段。

不能简单认为建了 `(a, b)` 就一定能高效支持所有 `a`、`b` 组合查询。

---

### 34. 什么是部分索引？

参考回答：

部分索引只给满足条件的一部分数据建立索引。

例如软删除用户，只查询未删除用户：

```sql
CREATE INDEX users_active_email_idx
ON app.users (email)
WHERE deleted_at IS NULL;
```

这样索引更小，查询活跃用户时更高效。

适合场景：

- 只查询未删除数据。
- 只查询某种状态的数据。
- 某些条件占总数据比例较小。

---

### 35. 为什么有索引但查询没有走索引？

参考回答：

可能原因很多：

- 表太小，顺序扫描更便宜。
- 查询返回大部分数据，走索引反而不划算。
- 查询条件和索引列不匹配。
- 对索引列做了函数或类型转换。
- 统计信息过旧。
- 组合索引列顺序不适合。
- `LIKE '%xxx'` 这种前置通配无法使用普通 B-Tree 索引。

所以不能只看“有没有索引”，要用：

```sql
EXPLAIN ANALYZE
```

看实际执行计划。

---

### 36. EXPLAIN 和 EXPLAIN ANALYZE 有什么区别？

参考回答：

`EXPLAIN` 只显示数据库计划怎么执行 SQL，不真正执行。

```sql
EXPLAIN SELECT * FROM users WHERE email = 'a@example.com';
```

`EXPLAIN ANALYZE` 会真正执行 SQL，并显示实际耗时和实际行数。

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'a@example.com';
```

对 `UPDATE`、`DELETE` 使用 `EXPLAIN ANALYZE` 要谨慎，因为它真的会修改数据。

可以放在事务里再回滚：

```sql
BEGIN;
EXPLAIN ANALYZE UPDATE users SET name = 'x' WHERE id = 1;
ROLLBACK;
```

---

### 37. Seq Scan、Index Scan、Bitmap Index Scan 分别是什么？

参考回答：

`Seq Scan` 是顺序扫描，数据库从头到尾扫描表。

`Index Scan` 是索引扫描，通过索引定位数据，再回表读取。

`Bitmap Index Scan` 通常用于查询命中较多行时，数据库先用索引构建位图，再批量访问表数据。

不能简单说 `Seq Scan` 一定差。小表或返回大量数据时，顺序扫描可能更快。

判断性能要结合：

- 表大小。
- 查询条件。
- 返回行数。
- 实际耗时。
- buffer 读取情况。

---

### 38. 索引有什么缺点？

参考回答：

索引的缺点：

- 占用磁盘空间。
- 插入、更新、删除时要维护索引，增加写入成本。
- 索引太多会影响写性能。
- 查询优化器选择计划时也要考虑更多索引。

所以索引不是越多越好。

应该根据高频查询、慢查询和执行计划来建立索引。

---

## 八、PostgreSQL 特色能力

### 39. uuid 类型适合什么场景？

参考回答：

`uuid` 适合作为公开资源 ID。

优点：

- 不容易被猜到。
- 不暴露数据增长趋势。
- 可以由数据库或应用生成。
- 适合分布式场景。

示例：

```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid()
```

短链接项目中，`id` 可以用 UUID，`code` 作为业务短码。

---

### 40. json 和 jsonb 有什么区别？

参考回答：

`json` 保存原始 JSON 文本。

`jsonb` 保存解析后的二进制格式，更适合查询和索引。

PostgreSQL 项目中通常更推荐 `jsonb`，因为它支持更好的查询和 GIN 索引。

适合存储：

- 用户偏好设置。
- 扩展配置。
- 结构不完全固定的数据。

但不应该把 PostgreSQL 当 MongoDB 用。如果数据结构稳定、关系清晰、需要强约束，优先使用普通关系表。

---

### 41. JSONB 适合存什么？不适合存什么？

参考回答：

适合：

- 用户偏好配置。
- 第三方回调原始数据。
- 扩展字段。
- 不同类型对象有少量差异字段。

不适合：

- 核心业务强约束字段。
- 需要频繁 JOIN 的关系数据。
- 需要外键约束的数据。
- 经常按多个字段复杂查询的数据。

例如用户昵称、邮箱、状态应该放普通列；用户 UI 设置可以放 `jsonb`。

---

### 42. GIN 索引是什么？

参考回答：

GIN 是 PostgreSQL 的一种索引类型，适合一个字段里包含多个值的情况。

常见用于：

- `jsonb`
- array
- 全文搜索 `tsvector`

例如：

```sql
CREATE INDEX users_settings_gin_idx
ON app.users
USING gin (settings);
```

可以加速 JSONB 包含查询。

---

### 43. Array 类型应该怎么用？

参考回答：

PostgreSQL 支持数组类型，例如：

```sql
tags text[]
```

适合存储简单、数量较少、不需要强关系约束的列表。

但如果这些数据需要：

- 频繁查询。
- 统计。
- 外键约束。
- 维护关系。

通常应该使用关联表，而不是数组。

例如文章标签，如果标签本身有独立属性，最好使用：

```text
posts
tags
post_tags
```

---

## 九、Go 接入 PostgreSQL

### 44. pgx 和 database/sql 有什么区别？

参考回答：

`database/sql` 是 Go 标准库提供的通用数据库接口，不绑定具体数据库。

`pgx` 是专门为 PostgreSQL 设计的驱动和工具库，对 PostgreSQL 支持更直接，功能更丰富。

如果项目明确使用 PostgreSQL，可以优先使用 `pgx` 和 `pgxpool`。

但了解 `database/sql` 也有价值，因为很多 Go 数据库生态都围绕它。

---

### 45. pgxpool 是什么？

参考回答：

`pgxpool` 是 pgx 提供的连接池。

它负责复用数据库连接，避免每个请求都新建连接。

使用连接池可以：

- 减少连接创建成本。
- 控制最大连接数。
- 提高并发访问数据库的稳定性。

但连接池不是越大越好，要结合 PostgreSQL 的 `max_connections` 和应用实例数量配置。

---

### 46. 为什么要使用参数化查询？

参考回答：

参数化查询可以避免 SQL 注入，也能让 SQL 和参数分离。

错误写法：

```go
sql := "SELECT * FROM users WHERE email = '" + email + "'"
```

正确写法：

```go
row := pool.QueryRow(ctx, `
    SELECT id, email
    FROM app.users
    WHERE email = $1
`, email)
```

用户输入永远不要直接拼接进 SQL。

---

### 47. QueryRow 和 Query 有什么区别？

参考回答：

`QueryRow` 用于查询单行结果。

例如根据 ID 查询用户：

```go
err := pool.QueryRow(ctx, query, id).Scan(&user.ID, &user.Email)
```

`Query` 用于查询多行结果，需要遍历 `rows`。

```go
rows, err := pool.Query(ctx, query)
if err != nil {
    return err
}
defer rows.Close()

for rows.Next() {
    // scan
}

if err := rows.Err(); err != nil {
    return err
}
```

查询多行时一定要关闭 `rows`，并检查 `rows.Err()`。

---

### 48. 如何处理唯一约束冲突？

参考回答：

PostgreSQL 唯一约束冲突的 SQLSTATE 是 `23505`。

使用 pgx 可以判断 `*pgconn.PgError`：

```go
var pgErr *pgconn.PgError
if errors.As(err, &pgErr) && pgErr.Code == "23505" {
    return ErrEmailAlreadyExists
}
```

在用户模块中，邮箱重复时应该返回明确业务错误，而不是直接把数据库错误返回给用户。

短链接项目中，短码冲突时可以重新生成 code 并重试。

---

## 十、数据库迁移

### 49. 什么是数据库迁移？

参考回答：

数据库迁移是用版本化文件管理数据库结构变化。

例如：

- 创建表。
- 新增字段。
- 创建索引。
- 修改约束。

迁移文件会提交到 Git，让团队和不同环境按同一套变更升级数据库。

这样比手动去生产执行 `ALTER TABLE` 更可控、可审查、可复现。

---

### 50. up/down 迁移是什么意思？

参考回答：

`up` 表示向前升级数据库结构。

`down` 表示回滚这个变更。

例如 up：

```sql
ALTER TABLE app.users ADD COLUMN avatar_url text;
```

down：

```sql
ALTER TABLE app.users DROP COLUMN avatar_url;
```

但要注意，down 不是万能的。如果删除字段或删除数据，即使 down 把字段加回来，原来的数据也回不来。

---

### 51. 已经上线的迁移文件可以修改吗？

参考回答：

不建议修改已经上线或已经被别人执行过的迁移文件。

原因是不同环境可能已经记录了这个迁移版本，如果你偷偷修改旧文件，会导致环境之间不一致。

正确做法是新增一个迁移文件继续变更。

学习时可以改，真实团队项目里要谨慎。

---

### 52. 生产迁移要注意什么？

参考回答：

生产迁移要注意：

- 是否会锁表。
- 是否涉及大表。
- 是否会长时间执行。
- 是否删除数据。
- 是否能回滚。
- 是否和当前应用代码兼容。
- 是否已经备份。
- 是否在测试环境验证过。

新增 nullable 字段通常风险较低；删除字段、修改字段类型、大表创建索引、大批量更新风险更高。

生产大表创建索引可以考虑：

```sql
CREATE INDEX CONCURRENTLY ...
```

但要注意迁移工具是否支持非事务迁移。

---

## 十一、生产实践与排查

### 53. 数据库连接数满了怎么办？

参考回答：

先查看最大连接数：

```sql
SHOW max_connections;
```

查看当前连接：

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state;
```

再按用户和应用分组：

```sql
SELECT usename, application_name, state, count(*)
FROM pg_stat_activity
GROUP BY usename, application_name, state
ORDER BY count(*) DESC;
```

常见原因：

- 应用连接池太大。
- 服务实例太多。
- 慢查询占住连接。
- 事务没关闭。
- 连接泄露。

解决方向：

- 调整连接池。
- 增加超时。
- 优化慢 SQL。
- 排查 `idle in transaction`。
- 必要时使用 PgBouncer。

---

### 54. 如何查看正在执行的 SQL？

参考回答：

使用 `pg_stat_activity`：

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS running_time,
    query
FROM pg_stat_activity
WHERE datname = current_database()
ORDER BY running_time DESC;
```

可以看到当前连接状态、运行时间、等待事件和 SQL。

这是排查慢查询、锁等待和长事务的重要入口。

---

### 55. idle in transaction 是什么？为什么危险？

参考回答：

`idle in transaction` 表示事务已经开启，但当前没有执行 SQL，也没有提交或回滚。

它危险是因为：

- 可能持有锁。
- 可能阻止 VACUUM 清理旧版本。
- 可能导致表膨胀。
- 长时间占用连接。

Go 代码里要用：

```go
defer tx.Rollback(ctx)
```

确保出错时事务能关闭。

---

### 56. VACUUM 和 ANALYZE 分别做什么？

参考回答：

`VACUUM` 用于清理 MVCC 产生的旧行版本，让空间可以复用，并维护数据库健康。

`ANALYZE` 用于收集统计信息，让查询优化器能选择更合理的执行计划。

PostgreSQL 默认有 autovacuum 自动执行这些工作。

但长事务、高频更新、大量删除都可能影响清理效果。

---

### 57. 备份和恢复怎么做？

参考回答：

可以使用 `pg_dump` 做逻辑备份。

推荐 custom 格式：

```powershell
pg_dump -Fc -f backup.dump $env:DATABASE_URL
```

恢复：

```powershell
pg_restore -d $env:RESTORE_DATABASE_URL backup.dump
```

重要的是不要只备份，还要做恢复演练。没有验证过恢复的备份不能算真正可靠。

---

### 58. 应用账号为什么不应该用 postgres 超级用户？

参考回答：

`postgres` 通常是超级用户，权限太大。

如果应用配置泄露或代码有 bug，可能造成严重破坏，例如删除表、创建高权限用户、修改系统配置等。

生产项目应该创建最小权限应用账号，只授予业务所需的 `SELECT`、`INSERT`、`UPDATE`、`DELETE` 等权限。

迁移账号和应用账号也最好分开。

---

## 十二、项目设计类高频题

### 59. 如果让你设计一个用户表，你会怎么设计？

参考回答：

可以设计：

```sql
CREATE TABLE app.users (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email text NOT NULL,
    username text NOT NULL,
    password_hash text NOT NULL,
    display_name text,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    deleted_at timestamptz,
    CONSTRAINT users_email_unique UNIQUE (email),
    CONSTRAINT users_username_unique UNIQUE (username),
    CONSTRAINT users_email_not_empty_check CHECK (length(trim(email)) > 0),
    CONSTRAINT users_username_not_empty_check CHECK (length(trim(username)) > 0)
);
```

解释：

- `id` 用 uuid，适合作为公开用户 ID。
- `email` 唯一，防止重复注册。
- 密码不存明文，存 hash。
- `created_at`、`updated_at` 用 `timestamptz`。
- `deleted_at` 支持软删除。
- 用约束保证关键字段合法。

---

### 60. 如果让你设计短链接表，你会怎么设计？

参考回答：

可以设计：

```sql
CREATE TABLE app.short_links (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    code text NOT NULL,
    original_url text NOT NULL,
    visit_count bigint NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT short_links_code_unique UNIQUE (code),
    CONSTRAINT short_links_code_not_empty_check CHECK (length(trim(code)) > 0),
    CONSTRAINT short_links_original_url_not_empty_check CHECK (length(trim(original_url)) > 0),
    CONSTRAINT short_links_visit_count_non_negative_check CHECK (visit_count >= 0)
);

CREATE INDEX short_links_created_at_desc_idx
ON app.short_links (created_at DESC);
```

解释：

- `id` 是主键。
- `code` 是短码，必须唯一。
- `original_url` 保存原始链接。
- `visit_count` 记录访问次数。
- `created_at` 用于列表排序。
- `code` 唯一约束会自动创建唯一索引，用于跳转查询。

---

### 61. 短链接 code 冲突怎么办？

参考回答：

不能只靠应用层先查再插入，因为并发下会有竞态。

正确做法是：

```text
数据库给 code 加 UNIQUE 约束。
应用生成 code 后尝试插入。
如果遇到唯一约束冲突 23505，就重新生成并重试。
```

这样数据库负责最终兜底。

---

### 62. 短链接访问次数怎么更新才不会丢失？

参考回答：

不要先查出 `visit_count` 到 Go 里加一，再写回。

错误思路：

```text
SELECT visit_count
Go 里 +1
UPDATE visit_count = 新值
```

并发下可能丢失更新。

正确写法：

```sql
UPDATE app.short_links
SET visit_count = visit_count + 1,
    updated_at = now()
WHERE code = $1
RETURNING original_url;
```

这个加法在数据库内部完成，PostgreSQL 会通过行级锁保证并发更新同一行时不会简单覆盖。

---

### 63. 如何设计订单和订单明细？

参考回答：

订单通常拆成订单主表和订单明细表。

```sql
CREATE TABLE app.orders (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id uuid NOT NULL,
    status text NOT NULL,
    total_amount bigint NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE app.order_items (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id uuid NOT NULL REFERENCES app.orders(id),
    product_id uuid NOT NULL,
    quantity integer NOT NULL,
    price bigint NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

说明：

- 订单主表保存订单整体信息。
- 明细表保存每个商品。
- 金额建议用整数保存最小货币单位，比如分，不建议用浮点数。
- 创建订单和订单明细通常需要事务。

---

### 64. 库存扣减怎么避免超卖？

参考回答：

一种简单方式是用条件更新：

```sql
UPDATE app.products
SET stock = stock - $1
WHERE id = $2
  AND stock >= $1;
```

然后检查影响行数。

如果影响行数是 0，说明库存不足。

这个更新在数据库内部是原子的，可以避免并发下库存扣成负数。

如果业务更复杂，也可以使用事务加 `SELECT ... FOR UPDATE` 锁住商品行，再检查库存和扣减。

---

## 十三、面试表达技巧

### 65. 不会的问题怎么回答？

参考回答方式：

不要硬编。

可以说：

```text
这个点我没有在项目里实际用过，但我知道大概方向是……
如果让我排查，我会先看……
```

例如问到主从复制细节，如果没学深：

```text
我目前对主从复制只了解基本概念，知道它可以用于读扩展和高可用，也知道读写分离会有复制延迟。具体搭建和故障切换我还没有实战过。
```

这样比乱答更可信。

---

### 66. 如何把短链接项目讲得像样？

可以这样讲：

```text
我做过一个 Go + PostgreSQL 的短链接服务。核心表是 short_links，用 uuid 做主键，code 做唯一短码，original_url 存原始链接，visit_count 记录访问次数。

创建短链接时，服务端用 crypto/rand 生成短码，然后插入数据库。code 上有唯一约束，如果发生 23505 冲突，就重新生成并重试。

跳转时使用 UPDATE ... RETURNING，一条 SQL 完成 visit_count + 1 并返回 original_url，避免并发下访问次数丢失。

查询详情按 code 查，因为 code 有唯一索引，所以查询效率比较稳定。列表按 created_at DESC 排序，加了 created_at 的索引。

项目里使用 pgxpool 管理连接池，SQL 都用参数化查询，表结构用迁移文件管理。
```

这段非常适合面试中回答“你项目里怎么用 PostgreSQL 的”。

---

### 67. 数据库面试回答的通用结构

建议按这个结构回答：

```text
先说概念。
再说为什么需要它。
再举一个 SQL 或项目例子。
最后说注意点。
```

例如回答索引：

```text
索引是数据库加速查询的数据结构，类似目录。比如用户邮箱登录时，可以给 email 建唯一索引，这样 WHERE email = $1 查询会更快，同时还能保证邮箱唯一。但索引会增加写入和存储成本，所以不能乱加，要结合 EXPLAIN 看执行计划。
```

这个结构清楚、有例子、有取舍，面试官会更容易认可。

---

## 十四、实习面试重点优先级

如果时间有限，优先掌握这些：

1. 基础 SQL：`SELECT`、`INSERT`、`UPDATE`、`DELETE`。
2. 表设计：主键、外键、唯一约束、非空约束。
3. JOIN：`INNER JOIN`、`LEFT JOIN`。
4. 聚合：`GROUP BY`、`HAVING`。
5. 事务：`BEGIN`、`COMMIT`、`ROLLBACK`、ACID。
6. 并发：脏读、不可重复读、幻读、MVCC。
7. 索引：B-Tree、组合索引、索引失效、`EXPLAIN`。
8. Go 接入：`pgxpool`、参数化查询、错误处理。
9. 迁移：up/down、为什么不要手动改生产表。
10. 项目：短链接表设计、短码冲突、访问计数原子更新。

这些掌握好，已经能覆盖大多数后端实习数据库面试。

---

## 十五、最后的复习清单

面试前你应该能不看资料回答：

- PostgreSQL 和 MySQL 有什么区别？
- 主键、外键、唯一约束分别是什么？
- `INNER JOIN` 和 `LEFT JOIN` 有什么区别？
- `WHERE` 和 `HAVING` 有什么区别？
- 什么是事务？
- ACID 分别是什么？
- 什么是 MVCC？
- PostgreSQL 默认隔离级别是什么？
- 什么是索引？
- 组合索引为什么顺序重要？
- 为什么有索引但不一定走索引？
- `EXPLAIN ANALYZE` 有什么用？
- `pgxpool` 是什么？
- 为什么要参数化查询？
- 如何处理唯一约束冲突？
- 什么是数据库迁移？
- 生产迁移要注意什么？
- 如何设计用户表？
- 如何设计短链接表？
- 短码冲突怎么处理？
- 访问次数并发更新怎么保证正确？

如果这些问题你能讲清楚，就已经具备后端实习数据库部分的竞争力。

