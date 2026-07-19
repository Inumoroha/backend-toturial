# TSDB、保留和容量估算

## 1. 本地 TSDB 的直觉模型

Prometheus 不是把每个样本写成一行数据库记录。它按时间序列组织样本，内存中的 head 和 WAL 保障近期写入，后台把数据压缩成 block。

你需要知道的不是实现细节，而是这些关系：

```text
series 数量增加 -> 内存、索引和查询成本增加
sample rate 增加 -> WAL、压缩、磁盘和远程写入成本增加
保留时间增加 -> 本地磁盘占用增加
查询时间范围/并发增加 -> 查询 CPU 和内存增加
```

## 2. 关键自监控指标

不同版本名称可能略有变化，先在自身 `/metrics` 中确认：

```promql
prometheus_tsdb_head_series
prometheus_tsdb_head_samples_appended_total
rate(prometheus_tsdb_head_samples_appended_total[5m])
prometheus_tsdb_wal_fsync_duration_seconds
prometheus_tsdb_compactions_failed_total
prometheus_engine_query_duration_seconds
prometheus_engine_query_failures_total
```

抓取侧：

```promql
sum by (job) (scrape_samples_scraped)
max by (job) (scrape_duration_seconds)
sum by (job) (rate(scrape_samples_post_metric_relabeling[5m]))
```

## 3. 粗略估算

要收集：

```text
target 数量
每个 target 的 series 数
scrape interval
每秒样本数
保留时间
峰值查询并发
远程写入副本数
```

样本速率的粗略关系：

```text
samples_per_second ≈ total_series / scrape_interval_seconds
```

这是第一版估算，不等于磁盘大小。压缩、label 长度、block 和 WAL 都会影响实际结果。

一个可复核的示例：

```text
1000 targets × 500 series/target = 500,000 series
500,000 / 15s ≈ 33,333 samples/s
```

这还没有加入规则输出、Histogram bucket、HA 副本和抓取峰值。容量文档至少要按低峰、平均、发布/扩容峰值分别估算，并预留磁盘和内存安全余量。

## 4. 实验测量

1. 启动单个 app，记录 `head_series`。
2. 加入一个有限 label，记录增长。
3. 加入一个用户 ID label，生成 10000 个值。
4. 观察内存、WAL、查询和 scrape 样本。
5. 删除高基数指标并重启，观察恢复时间。

实验记录必须包含时间点和版本，否则无法比较前后结果。

## 5. 保留策略

按时间：

```text
保留 15d
```

按大小：

```text
保留到磁盘使用达到某个上限
```

生产通常需要同时考虑时间和磁盘安全边界。保留期越长不一定越好；长期趋势和明细排障可以交给长期存储，Prometheus 本地保留更适合近期高分辨率查询。

## 6. 容量文档模板

```text
环境：
Prometheus 版本：
target 数量：
平均每 target series：
scrape interval：
估算 samples/s：
本地保留：
磁盘预算：
内存预算：
查询并发假设：
远程写入：
压测结果：
安全余量：
```

## 7. 阶段验收

给出一份“1000 targets、平均 500 series、15s 抓取、15 天保留”的初步方案。明确哪些数值是估算、哪些必须压测验证，并列出扩容触发条件。

同时提交一次测量记录：head series、samples/s、WAL 磁盘增长、查询 P95 和 Prometheus 内存。理论估算与实际差异超过 30% 时，必须说明差异来源。
