# 阶段 1：Prometheus 入门

这一阶段把 Prometheus 从“一个容器”变成你能解释、能排障的监控系统。

## 学习目标

- 画出 target、scrape、sample、time series 和 TSDB 的关系。
- 启动 Prometheus，并让一个 Go 服务被抓取。
- 使用 `/targets`、`/rules`、`/alerts` 和 HTTP API。
- 区分抓取失败、应用错误和查询无数据。
- 学会修改配置、验证配置和观察重启后的行为。

## 教程顺序

1. [架构、数据模型与抓取模型](01-architecture-data-model.md)
2. [Docker Compose 实验环境](02-docker-compose-lab.md)
3. [Target、抓取与排障](03-scrape-and-target-debugging.md)

## 交付物

- 一个可以启动和销毁的本地 Prometheus 环境。
- 一份 `prometheus.yml`，包含 Prometheus 自监控和 Go 服务 target。
- 一次 target 断开、路径错误和格式错误的故障记录。
- 通过 API 查询 `up` 并保存 JSON 结果。

## 阶段验收

不看教程完成以下操作：

```text
启动 -> 查看 targets -> 查询 up -> 停止 app -> 解释 up 变化
-> 修复 target -> 确认恢复 -> 删除卷 -> 重新启动
```

注意：这一阶段先不急着写复杂 PromQL。先把“数据有没有进入 Prometheus”这件事搞清楚。

