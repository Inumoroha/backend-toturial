# 阶段 8：订单平台可观测性综合项目

这是全路线的毕业项目。你要交付的不是一张 Dashboard，而是一套从请求到告警、从故障到复盘都能被复现的系统。

## 项目目标

实现：

```text
API Gateway -> Order API -> Inventory / Payment 模拟依赖
                    |
                    +-> Queue Worker -> PostgreSQL/Redis（可选）
```

推荐先用内存或 HTTP 模拟依赖，完成可观测性后再接入数据库和消息队列。不要一开始把业务复杂度变成监控复杂度。

## 教程顺序

1. [项目需求、架构与指标契约](01-project-requirements.md)
2. [故障注入、告警演练与 Runbook](02-fault-injection-runbooks.md)
3. [验收、复盘与面试表达](03-acceptance-interview.md)

## 必须交付

- 2～3 个 Go 服务：Order API、Worker、至少一个模拟依赖。
- Docker Compose 一键启动。
- Prometheus 配置、Recording Rules、Alerting Rules。
- Alertmanager 路由和通知适配器。
- Grafana 总览、服务、依赖/队列和资源 Dashboard。
- 至少 5 个故障注入场景。
- 每条 page 告警对应 Runbook。
- 3 次事故复盘。
- README、架构图、容量估算和已知限制。

## 推荐里程碑

| 里程碑 | 证明 |
| --- | --- |
| M1 服务可运行 | API 和 Worker 有测试，依赖可模拟 |
| M2 指标可抓取 | targets 全 UP，指标设计表完成 |
| M3 查询可排障 | RED、依赖、队列查询可用 |
| M4 告警可行动 | 规则测试、通知、Runbook 完成 |
| M5 故障可复现 | 延迟、5xx、积压、重启和抓取失败演练 |
| M6 生产化 | 容量、HA、安全、升级和恢复方案 |
| M7 作品集 | README、截图、复盘和面试讲解 |

## 项目仓库建议

```text
order-observability/
├── services/
│   ├── order-api/
│   ├── order-worker/
│   └── fake-dependency/
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/recording.yml
│   ├── rules/alerts.yml
│   └── tests/
├── alertmanager/
├── grafana/
├── fault-injection/
├── runbooks/
├── deploy/docker-compose.yml
├── deploy/k8s/
└── README.md
```

## 完成标准

陌生工程师拿到仓库后，能在 15 分钟内启动环境；你能在 10 分钟内从告警和 Dashboard 定位一次注入的错误；清理环境后能重新得到相同的结果。

