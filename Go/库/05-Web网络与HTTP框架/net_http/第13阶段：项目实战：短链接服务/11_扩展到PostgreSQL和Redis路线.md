# 11. 扩展到 PostgreSQL 和 Redis 路线

本节目标：理解短链接服务从内存版扩展到真实后端存储时，哪些模块要改，哪些设计可以复用。

---

## 一、为什么内存版不够

内存版适合学习，但有明显限制：

- 服务重启数据丢失。
- 无法多实例共享数据。
- 不适合长期保存访问统计。
- 内存容量有限。

真实短链接服务至少需要持久化存储。

---

## 二、PostgreSQL 表设计方向

核心表：

```sql
CREATE TABLE links (
    id bigserial PRIMARY KEY,
    code text NOT NULL UNIQUE,
    original_url text NOT NULL,
    visit_count bigint NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

关键点：

- `code` 必须有唯一约束。
- 跳转查询依赖 `code` 索引。
- 访问计数用原子更新。

---

## 三、Store 接口可以保持不变

前面定义了：

```go
type Store interface {
	Create(ctx context.Context, item Link) (Link, error)
	GetByCode(ctx context.Context, code string) (Link, error)
	IncrementVisit(ctx context.Context, code string) error
	List(ctx context.Context) ([]Link, error)
}
```

换 PostgreSQL 时，只需要新增：

```go
type PostgresStore struct {
	pool *pgxpool.Pool
}
```

Service 和 Handler 可以尽量不改。

---

## 四、Redis 可以做什么

Redis 常见用途：

- 缓存 `code -> original_url`。
- 热门短码加速跳转。
- 访问次数先写 Redis，再异步落库。
- 做简单限流。

但不要一开始就加 Redis。先把 PostgreSQL 版本做稳，再基于性能瓶颈决定是否加缓存。

---

## 五、访问计数的设计选择

简单一致版本：

```text
每次跳转同步 UPDATE PostgreSQL。
```

优点：实现简单，数据较准。  
缺点：高流量下数据库写压力大。

异步统计版本：

```text
跳转时快速返回。
访问事件写队列或 Redis。
后台批量聚合落库。
```

优点：跳转快，抗流量。  
缺点：系统复杂，统计最终一致。

---

## 六、阶段复盘

你应该能回答：

- 为什么内存 Store 可以被 PostgreSQL Store 替换？
- 数据库中哪个约束保证短码唯一？
- 为什么 `visit_count = visit_count + 1` 是关键写法？
- Redis 适合加在哪里？
- 为什么不要一开始就上复杂架构？
