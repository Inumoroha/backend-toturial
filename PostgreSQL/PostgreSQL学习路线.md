# PostgreSQL + Go 后端学习路线图（综合版）

这份路线图面向“想把 PostgreSQL 用到 Go 后端项目里”的学习目标。它融合了原路线中适合入门的实践导向，也补充了真实项目里更重要的表设计、查询能力、事务、索引、迁移和性能分析。

学习顺序建议是：先建立 SQL 和关系建模基础，再学习 PostgreSQL 的特色能力，最后接入 Go 项目并完成一个可落地的小项目。

---

## 第一阶段：环境搭建与数据库直觉

目标：能启动 PostgreSQL，连接数据库，并理解数据库、表、字段、记录之间的关系。

### 需要掌握

- 使用 Docker 启动 PostgreSQL，避免污染本地环境。
- 使用 `psql`、DBeaver 或 DataGrip 连接数据库。
- 理解数据库、schema、table、row、column 的基本概念。
- 理解 PostgreSQL 与 MySQL、SQLite、MongoDB 的大致区别。

### 推荐实践

- 启动一个 PostgreSQL 容器。
- 创建一个测试数据库，例如 `learn_pg`。
- 使用图形化客户端连接数据库。
- 用 `psql` 手动执行几条 SQL，熟悉命令行交互。

### 注意点

一开始不要急着研究底层存储、B-Tree、MVCC 等原理。先学会和数据库“对话”，把 SQL 写顺手。

---

## 第二阶段：SQL 基础与单表操作

目标：能完成最基础的建表、插入、查询、修改和删除。

### 需要掌握

- DDL：`CREATE TABLE`、`ALTER TABLE`、`DROP TABLE`。
- DML：`INSERT`、`SELECT`、`UPDATE`、`DELETE`。
- 条件查询：`WHERE`。
- 排序：`ORDER BY`。
- 分页：`LIMIT`、`OFFSET`。
- 常见操作符：`=`、`<>`、`>`、`<`、`>=`、`<=`、`LIKE`、`IN`、`BETWEEN`、`IS NULL`。

### 推荐实践

创建一张用户表：

```sql
CREATE TABLE users (
    id bigserial PRIMARY KEY,
    name text NOT NULL,
    email text NOT NULL UNIQUE,
    age integer,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

然后练习：

- 插入几条用户数据。
- 查询年龄大于某个值的用户。
- 按创建时间倒序排列。
- 更新某个用户的邮箱。
- 删除测试数据。

### 注意点

这个阶段重点不是背 SQL，而是形成直觉：数据库中的数据不是“文件里的文本”，而是有类型、有约束、可查询、可组合的数据集合。

---

## 第三阶段：表设计与关系建模

目标：能设计结构清晰、约束合理的数据表。

### 需要掌握

- 主键：`PRIMARY KEY`。
- 外键：`FOREIGN KEY`。
- 唯一约束：`UNIQUE`。
- 非空约束：`NOT NULL`。
- 检查约束：`CHECK`。
- 默认值：`DEFAULT`。
- 常见字段类型：`integer`、`bigint`、`text`、`varchar`、`boolean`、`timestamp`、`timestamptz`、`uuid`。
- 一对一、一对多、多对多关系。

### 推荐实践

设计一个简单博客系统的数据表：

- `users`：用户表。
- `posts`：文章表，一个用户可以有多篇文章。
- `comments`：评论表，一篇文章可以有多条评论。
- `tags`：标签表。
- `post_tags`：文章和标签的多对多关联表。

### 注意点

关系型数据库最重要的能力不是“存数据”，而是用清晰的结构表达业务关系。JSONB 和 Array 很强，但不要用它们逃避正常的关系建模。

---

## 第四阶段：查询能力进阶

目标：能写出真实业务中常见的复杂查询。

### 需要掌握

- 聚合函数：`COUNT`、`SUM`、`AVG`、`MAX`、`MIN`。
- 分组查询：`GROUP BY`、`HAVING`。
- 多表查询：`INNER JOIN`、`LEFT JOIN`。
- 子查询。
- 去重：`DISTINCT`。
- 条件表达式：`CASE WHEN`。
- 插入或更新：`INSERT ... ON CONFLICT ... DO UPDATE`。

### 推荐实践

基于博客系统练习：

- 查询每个用户发布了多少篇文章。
- 查询文章列表，同时显示作者名称。
- 查询没有评论的文章。
- 查询评论数最多的前 10 篇文章。
- 使用 `ON CONFLICT` 实现“邮箱已存在则更新用户信息”。

### 注意点

`JOIN` 不应该放得太晚。对后端开发来说，多表查询是基础能力，不是性能优化阶段才需要接触的内容。

---

## 第五阶段：事务、并发与数据一致性

目标：理解 PostgreSQL 如何保证数据可靠，并能在业务代码中正确使用事务。

### 需要掌握

- 事务语句：`BEGIN`、`COMMIT`、`ROLLBACK`。
- ACID：原子性、一致性、隔离性、持久性。
- 常见并发问题：脏读、不可重复读、幻读。
- PostgreSQL 的 MVCC 基本思想。
- 事务隔离级别。
- 常见锁问题。
- 长事务的风险。

### 推荐实践

实现一个简单转账场景：

- A 用户余额减少。
- B 用户余额增加。
- 两个操作必须同时成功或同时失败。
- 故意制造错误，观察 `ROLLBACK` 的效果。

### 注意点

后端项目里很多严重 bug 都不是 SQL 写错，而是事务边界不清楚。比如库存扣减、订单创建、账户余额变更，都必须认真处理事务。

---

## 第六阶段：索引与性能分析

目标：能判断查询为什么慢，并知道如何优化。

### 需要掌握

- B-Tree 索引。
- 唯一索引。
- 组合索引。
- 部分索引。
- 索引失效的常见原因。
- `EXPLAIN`。
- `EXPLAIN ANALYZE`。
- 顺序扫描：`Seq Scan`。
- 索引扫描：`Index Scan`。
- 位图扫描：`Bitmap Index Scan`。

### 推荐实践

- 给 `users.email` 建唯一索引。
- 给 `posts.user_id` 建普通索引。
- 给 `posts.created_at` 建索引，优化按时间排序的查询。
- 使用 `EXPLAIN ANALYZE` 对比建索引前后的查询计划。

### 注意点

不要陷入“建了索引就一定更快”的误区。索引会提升查询速度，但也会增加写入成本和存储成本。是否有效，要看执行计划。

---

## 第七阶段：PostgreSQL 特色能力

目标：理解 PostgreSQL 相比普通关系型数据库更强的能力，并知道什么时候该用、什么时候不该用。

### 需要掌握

- `uuid` 类型。
- `jsonb` 类型。
- Array 类型。
- GIN 索引。
- 枚举类型。
- 范围类型。
- 全文搜索基础。

### 推荐实践

- 用 `uuid` 作为公开资源 ID。
- 用 `jsonb` 存储扩展配置，例如用户偏好设置。
- 给 `jsonb` 字段建立 GIN 索引。
- 对比普通关系表和 JSONB 在查询、约束、维护上的差异。

### 注意点

PostgreSQL 的 JSONB 很强，但不等于要把 PostgreSQL 当 MongoDB 用。如果数据结构稳定、关系清晰、需要强约束，优先使用普通表结构。

Array 也不是为了随意省掉关联表。需要查询、统计、约束和维护关系的数据，通常还是关联表更清晰。

---

## 第八阶段：Go 接入 PostgreSQL

目标：能在 Go 服务中稳定、清晰地操作 PostgreSQL。

### 需要掌握

- `github.com/jackc/pgx`。
- `pgxpool` 连接池。
- 参数化查询，避免 SQL 注入。
- 查询单行和多行数据。
- 插入、更新、删除。
- 在 Go 中使用事务。
- 处理数据库错误，例如唯一约束冲突、外键约束失败。
- 理解 `database/sql` 与 `pgx` 的区别。

### 推荐实践

用 Go 写一个用户模块：

- 创建用户。
- 根据 ID 查询用户。
- 根据邮箱查询用户。
- 更新用户信息。
- 删除用户。
- 用户邮箱重复时返回明确错误。

### 注意点

新项目如果明确使用 PostgreSQL，可以优先学习 `pgx` 和 `pgxpool`。但也建议知道 `database/sql` 是 Go 的通用数据库接口，这有助于理解 Go 的数据库生态。

---

## 第九阶段：工程化与数据库迁移

目标：用工程化方式管理数据库结构变化，而不是手动改表。

### 需要掌握

- 数据库迁移的意义。
- 迁移文件的 up/down 思想。
- `goose` 或 `golang-migrate`。
- 本地环境、测试环境、生产环境的迁移流程。
- 种子数据和测试数据的区别。

### 推荐实践

- 用迁移工具创建 `users` 表。
- 新增一个字段，例如 `avatar_url`。
- 回滚一次迁移。
- 再重新执行迁移。
- 把迁移命令写入项目 README 或 Makefile。

### 注意点

真实项目中，表结构会不断变化。如果没有迁移工具，团队协作和部署都会很痛苦。

---

## 第十阶段：项目落地：短链接服务

目标：把 PostgreSQL 真正接入 Go 后端项目，完成学习闭环。

### 项目功能

- 创建短链接。
- 根据短码跳转到原始长链接。
- 记录创建时间。
- 记录访问次数。
- 防止短码重复。
- 查询短链接详情。

### 推荐表结构

```sql
CREATE TABLE short_links (
    id uuid PRIMARY KEY,
    code text NOT NULL UNIQUE,
    original_url text NOT NULL,
    visit_count bigint NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
```

### 推荐接口

- `POST /links`：创建短链接。
- `GET /:code`：根据短码跳转。
- `GET /links/:code`：查询短链接详情。

### 需要用到的能力

- Go `net/http` 或常见 Web 框架。
- `pgxpool` 连接池。
- 参数化 SQL。
- 唯一索引。
- 事务。
- 迁移工具。
- `EXPLAIN ANALYZE` 分析核心查询。

### 可选增强

- 给短链接增加过期时间。
- 增加访问日志表。
- 统计每日访问量。
- 使用 Redis 缓存热门短链接。
- 给管理接口增加分页查询。

---

## 推荐学习节奏

如果每天学习 1 到 2 小时，可以按下面节奏推进：

1. 第 1 周：环境搭建、SQL 基础、单表 CRUD。
2. 第 2 周：表设计、约束、关系建模、JOIN。
3. 第 3 周：聚合、子查询、事务、并发基础。
4. 第 4 周：索引、执行计划、PostgreSQL 特色类型。
5. 第 5 周：Go + pgxpool 接入 PostgreSQL。
6. 第 6 周：迁移工具、短链接项目、性能检查。

如果时间更紧，可以先压缩成一条最小路线：

1. Docker 启动 PostgreSQL。
2. 学会基础 SQL 和表设计。
3. 学会 JOIN、事务、索引。
4. 学会 Go + `pgxpool`。
5. 做完短链接服务。

---

## 最终总结

这条综合路线的核心思路是：

1. 先会用 SQL 操作数据。
2. 再会设计表和表达业务关系。
3. 然后掌握 JOIN、事务、索引这些后端高频能力。
4. 接着学习 PostgreSQL 的 JSONB、UUID、Array、GIN 等特色能力。
5. 最后用 Go、`pgxpool`、迁移工具和短链接项目完成闭环。

学完之后，你的目标不是“知道 PostgreSQL 有哪些功能”，而是能够在真实 Go 后端项目中稳定地设计表、写查询、处理事务、分析慢 SQL，并把数据库结构变化纳入工程化管理。