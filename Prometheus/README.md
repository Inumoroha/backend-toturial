# Prometheus + Go 后端教程

这是一套沿着 [`prometheus-go-backend-learning-roadmap.md`](prometheus-go-backend-learning-roadmap.md) 落地的分阶段教程。目标不是记住几个命令，而是从一个真实的 Go 服务开始，逐步搭建可观测性、告警和生产运维能力。

## 学习顺序

| 阶段 | 目录 | 你会完成的事情 |
| --- | --- | --- |
| 0 | [Go 与基础能力](tutorial/00-foundations/README.md) | 写出可测试、可优雅退出、可诊断的 HTTP 服务 |
| 1 | [Prometheus 入门](tutorial/01-prometheus-basics/README.md) | 启动监控栈，理解数据模型和抓取排障 |
| 2 | [Go 埋点与指标设计](tutorial/02-go-instrumentation/README.md) | 为服务加入 RED、业务和依赖指标 |
| 3 | [PromQL](tutorial/03-promql/README.md) | 从原始样本写出速率、错误率、分位数和规则 |
| 4 | [告警](tutorial/04-alerting/README.md) | 编写规则、测试规则，并用 Alertmanager 路由通知 |
| 5 | [Grafana](tutorial/05-grafana/README.md) | 设计面向排障的 Dashboard |
| 6 | [Exporter 与 Kubernetes](tutorial/06-exporter-kubernetes/README.md) | 写 Exporter、服务发现并在集群中抓取 |
| 7 | [生产能力](tutorial/07-production/README.md) | 处理 TSDB、容量、HA、远程存储和安全 |
| 8 | [综合项目](tutorial/08-capstone/README.md) | 完成订单平台可观测性项目并进行故障演练 |

辅助模板在 [tutorial/templates](tutorial/templates/) 目录中，路线总览仍在根目录的学习路线文档中。

## 先做环境检查

建议在 Windows 上使用 Docker Desktop + WSL2，在 Linux/macOS 上直接使用 Docker Engine。生产命令默认以 Linux shell 语义说明，PowerShell 只需要调整路径和环境变量写法。

```bash
go version
docker version
docker compose version
git --version
curl --version
```

推荐版本原则：

- 使用 Go、Prometheus、Grafana 和 Alertmanager 的当前稳定版。
- 教程中的配置字段以你实际运行的官方版本文档为准。
- 每次升级都记录镜像 tag、配置差异和回滚方式，不要长期使用 `latest` 作为生产 tag。

## 建议的实验仓库

教程中的所有配置都建议保存到一个自己的实验仓库：

```text
prometheus-lab/
├── app/                  # Go 服务
├── prometheus/           # 主配置和规则
├── alertmanager/         # 路由配置
├── grafana/              # provisioning 和 Dashboard JSON
├── k8s/                  # Kubernetes manifests
├── runbooks/             # 告警排障手册
└── notes/                # 每周实验记录
```

每完成一个小节，都记录：

1. 使用了哪一个版本。
2. 执行了哪些命令。
3. 看到了什么现象。
4. 现象背后的原因是什么。
5. 如何验证恢复。
6. 哪个结论还没有被实验验证。

## 本教程的通用约定

- 指标名称使用小写、下划线和明确单位；时间用 `_seconds`，字节用 `_bytes`。
- 不把用户 ID、订单 ID、原始 URL、错误文本等无界值放入 label。
- 所有示例命令都先在本地实验环境执行；不要把示例密码、Webhook 或公网端口直接用于生产。
- 修改 YAML 后先运行 `promtool check config` 或对应组件的校验命令。
- 看到告警时先确认数据、时间窗口和 label 维度，再调整阈值。
- Dashboard、规则和 Runbook 都进入 Git，禁止只在 UI 中保存唯一副本。

## 阶段完成的最低证据

你可以在下一阶段开始前，用以下证据证明自己真的完成了当前阶段：

- 能不看教程复现启动命令。
- 能解释每一条配置的作用和失败时的表现。
- 能根据观察到的指标或日志提出至少一个可验证假设。
- 能删除并重新创建环境，仍然得到同样结果。
- 能把实验步骤写成别人可以执行的 README。

## 推荐学习节奏

每周安排五个单元：

- 理论：阅读一个概念主题，画出数据流。
- 编码：修改 Go 服务或配置。
- 查询：写 5～10 条 PromQL，并记录结果。
- 故障：主动制造一次抓取、延迟或错误问题。
- 复盘：更新 [每周学习记录模板](tutorial/templates/weekly-learning-log.md)。

不要连续看完所有章节再动手。每读完一节就执行其中的最小实验。
