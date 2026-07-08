# 第 8 阶段：分片与大规模数据

分片是 MongoDB 面向大规模数据和高吞吐写入的水平扩展能力。它不是“性能慢了就加分片”的万能药，而是当单个复制集在容量、吞吐或工作集上已经接近瓶颈时，用来把数据分布到多个 shard 上。

本阶段目标是让你能判断：什么时候需要分片，什么时候只是索引/建模/硬件问题；如果需要分片，如何选择 shard key，如何避免热点写入，如何设计日志/埋点系统。

## 本阶段目标

完成本阶段后，你应该能做到：

- 解释分片解决什么问题。
- 理解 shard、mongos、config server 的职责。
- 理解 shard key 为什么是分片系统最关键的设计。
- 理解 chunk、balancer 和数据迁移。
- 区分范围分片和哈希分片。
- 识别单调递增 shard key 的热点风险。
- 理解唯一索引和 shard key 的关系。
- 理解 targeted query 和 scatter-gather query。
- 设计一个千万级日志/埋点系统。
- 为大规模数据设计索引、TTL 和归档策略。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-什么时候需要分片.md` | 判断是否真的需要分片 |
| 02 | `02-分片集群组件-mongos-shard-config-server.md` | 学集群组件职责 |
| 03 | `03-shard-key设计原则.md` | 学 shard key 选择标准 |
| 04 | `04-范围分片与哈希分片.md` | 对比 ranged 和 hashed sharding |
| 05 | `05-chunk-balancer与数据迁移.md` | 学 chunk、balancer、迁移 |
| 06 | `06-热点写入与单调递增字段风险.md` | 识别和规避热点 |
| 07 | `07-唯一索引与分片键限制.md` | 学唯一约束和 shard key 的关系 |
| 08 | `08-查询路由-targeted与scatter-gather.md` | 学查询如何路由到 shard |
| 09 | `09-大规模日志埋点系统设计.md` | 综合设计日志/埋点模型 |
| 10 | `10-Go后端接入分片集群注意事项.md` | Go 连接 mongos 和工程注意事项 |
| 11 | `11-阶段练习-千万级事件系统方案.md` | 阶段综合练习 |
| 12 | `12-阶段验收与分片设计清单.md` | 验收和评审清单 |

## 本阶段核心判断

```text
先优化模型、索引、查询和硬件。
再判断是否需要分片。
一旦选择 shard key，后续变更成本很高。
```

## 官方资料

本阶段主要参考：

- Sharding：<https://www.mongodb.com/docs/manual/sharding/>
- Sharded Cluster Components：<https://www.mongodb.com/docs/manual/core/sharded-cluster-components/>
- Shard Keys：<https://www.mongodb.com/docs/manual/core/sharding-shard-key/>
- Hashed Sharding：<https://www.mongodb.com/docs/manual/core/hashed-sharding/>
- Ranged Sharding：<https://www.mongodb.com/docs/manual/core/ranged-sharding/>
- Chunks：<https://www.mongodb.com/docs/manual/core/sharding-data-partitioning/>
- Balancer：<https://www.mongodb.com/docs/manual/core/sharding-balancer-administration/>
- Unique Indexes in Sharded Clusters：<https://www.mongodb.com/docs/manual/core/sharded-cluster-requirements/#unique-indexes>

