# 05. RabbitMQ vs 数据库任务表

## 1. 数据库任务表是什么

数据库任务表是一种很朴素但实用的异步方案。

例如：

```sql
CREATE TABLE jobs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    job_type VARCHAR(64) NOT NULL,
    payload JSON NOT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    next_run_at TIMESTAMP NOT NULL,
    locked_at TIMESTAMP NULL,
    locked_by VARCHAR(64) NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

worker 定时扫描：

```text
pending jobs -> lock -> process -> done / retry / failed
```

## 2. 数据库任务表的优势

### 2.1 简单

不需要额外部署 RabbitMQ。

只要已有数据库，就能实现。

### 2.2 和业务事务天然一致

业务表和任务表在同一个数据库事务里写入。

例如：

```text
创建订单
写 orders
写 jobs(order_timeout_check)
提交事务
```

这天然解决：

```text
数据库成功但任务记录失败
```

### 2.3 易排查

任务都在数据库表里：

```sql
SELECT * FROM jobs WHERE status = 'failed';
```

对很多团队来说，数据库比消息中间件更容易理解。

## 3. 数据库任务表的劣势

### 3.1 吞吐有限

数据库不是专门消息队列。

高频扫描、锁竞争、大量更新会影响数据库。

### 3.2 实时性一般

worker 通常轮询：

```text
每 1 秒扫描一次
```

延迟通常高于消息推送。

### 3.3 路由能力弱

数据库任务表没有 RabbitMQ 的：

- exchange。
- binding。
- routing key。
- fanout。
- topic。

复杂订阅要自己实现。

### 3.4 容易拖累业务库

任务量大时，jobs 表会变热。

可能影响核心业务表。

## 4. 适合数据库任务表的场景

适合：

- 任务量小。
- 强依赖数据库事务。
- 只有一个或少数 worker。
- 业务可接受秒级延迟。
- 团队暂时没有消息中间件运维能力。
- 后台管理任务。
- 简单延迟任务。

例如：

```text
每天几千个报表生成任务
订单超时扫描
低频邮件任务
后台批处理
```

## 5. 不适合数据库任务表的场景

不适合：

- 高吞吐消息。
- 多系统复杂订阅。
- 低延迟业务事件。
- 大量 fanout。
- 复杂重试和 DLQ 治理。
- 希望独立扩展消息层。

这些更适合 RabbitMQ 或 Kafka。

## 6. 和 Outbox 的区别

Outbox 表不是普通任务表。

Outbox 的职责是：

```text
记录“待发布到消息中间件的事件”。
```

数据库任务表的职责是：

```text
直接让 worker 从数据库中取任务处理。
```

Outbox 后面还有 RabbitMQ。

数据库任务表本身就是队列。

## 7. 数据库任务表 worker 伪代码

```go
for {
	jobs := repo.LockPendingJobs(ctx, 100)
	for _, job := range jobs {
		err := handle(job)
		if err != nil {
			repo.MarkRetry(ctx, job.ID, err)
			continue
		}
		repo.MarkDone(ctx, job.ID)
	}
	time.Sleep(time.Second)
}
```

多 worker 场景要用：

```text
行锁
状态抢占
SKIP LOCKED
```

具体语法取决于数据库。

## 8. 怎么选

选择数据库任务表：

```text
任务量小
需要和数据库强一致
延迟要求不高
路由简单
团队想保持系统简单
```

选择 RabbitMQ：

```text
任务量较大
需要低延迟
多个系统订阅
需要复杂路由
需要独立消息层
```

## 9. 演进路线

很多系统可以这样演进：

```text
第 1 阶段：数据库任务表
第 2 阶段：Outbox + RabbitMQ
第 3 阶段：RabbitMQ + Kafka 分工
```

不要一开始就为了“架构高级”上复杂中间件。

## 10. 本节小结

数据库任务表不是低级方案。

它适合：

```text
低吞吐、强事务、简单任务、低运维成本。
```

RabbitMQ 适合：

```text
业务消息、复杂路由、任务分发、低延迟异步。
```

下一节学习 RabbitMQ 内部队列类型取舍。

