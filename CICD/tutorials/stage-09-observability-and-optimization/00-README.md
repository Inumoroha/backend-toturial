# 阶段 9：可观测性与流水线优化

> 本阶段目标：让 CI/CD 从“能跑”升级为“可观察、可度量、可优化、可持续改进”。你会学习如何定位流水线慢在哪里、失败为什么发生、发布后如何验证，以及如何用指标衡量交付效率。

## 学习顺序

请按下面顺序学习：

1. [01-observability-map-and-key-metrics.md](./01-observability-map-and-key-metrics.md)
   - 建立 CI/CD 可观测性地图。
   - 理解流水线、发布、应用运行时和团队交付指标之间的关系。

2. [02-workflow-logs-summaries-and-debugging.md](./02-workflow-logs-summaries-and-debugging.md)
   - 学习 workflow 日志、step summary、artifact 和失败排查。
   - 让 CI 输出对人更友好的诊断信息。

3. [03-pipeline-metrics-and-api-collection.md](./03-pipeline-metrics-and-api-collection.md)
   - 收集 workflow 运行时长、成功率、失败率、取消率和 job 耗时。
   - 使用 `gh` 和 GitHub API 建立简单指标采集。

4. [04-bottleneck-analysis-and-baseline.md](./04-bottleneck-analysis-and-baseline.md)
   - 建立优化前基线。
   - 分析 queue、setup、download、test、build、scan、deploy 的耗时瓶颈。

5. [05-go-cache-and-dependency-optimization.md](./05-go-cache-and-dependency-optimization.md)
   - 优化 Go module 和 build cache。
   - 学习 cache key、restore key、缓存命中率和缓存污染风险。

6. [06-docker-build-cache-and-image-optimization.md](./06-docker-build-cache-and-image-optimization.md)
   - 优化 Dockerfile 层顺序、`.dockerignore` 和 BuildKit 缓存。
   - 使用 buildx 的 GitHub Actions cache 加速镜像构建。

7. [07-parallelism-matrix-and-concurrency.md](./07-parallelism-matrix-and-concurrency.md)
   - 学习并行 job、matrix、`needs`、`concurrency` 和路径过滤。
   - 让 PR 反馈更快，同时避免生产部署并发冲突。

8. [08-test-strategy-and-flaky-failure-management.md](./08-test-strategy-and-flaky-failure-management.md)
   - 分层测试、失败分类和 flaky test 治理。
   - 防止“偶发失败”逐渐摧毁团队对 CI 的信任。

9. [09-deployment-observability-and-release-verification.md](./09-deployment-observability-and-release-verification.md)
   - 记录部署事件、镜像 digest、Argo CD 状态和冒烟测试结果。
   - 建立发布后验证流程。

10. [10-go-service-observability-prometheus-and-otel.md](./10-go-service-observability-prometheus-and-otel.md)
    - 给 Go 服务增加 `/metrics`、业务指标和追踪思路。
    - 理解 Prometheus、OpenTelemetry、日志和 trace 的边界。

11. [11-dashboards-alerts-and-slos.md](./11-dashboards-alerts-and-slos.md)
    - 设计 CI/CD 和应用发布 dashboard。
    - 学习告警、SLO、错误预算和告警降噪。

12. [12-dora-cost-and-delivery-reporting.md](./12-dora-cost-and-delivery-reporting.md)
    - 学习 DORA 指标：部署频率、变更前置时间、变更失败率、恢复时间。
    - 了解 runner 成本、artifact 保留、缓存治理和交付周报。

13. [13-practice-observe-and-optimize-go-cicd-lab.md](./13-practice-observe-and-optimize-go-cicd-lab.md)
    - 为 `go-cicd-lab` 建立可观测和优化实战。
    - 输出优化报告、发布证据和改进清单。

14. [14-review-checklist-and-quiz.md](./14-review-checklist-and-quiz.md)
    - 自测题和阶段验收标准。

## 本阶段建议时间

- 快速学习：1 周。
- 扎实学习：2 到 3 周。
- 推荐方式：先建立指标基线，再做缓存和并发优化，最后补齐发布验证和团队交付指标。

## 本阶段你要准备什么

- 已完成前面阶段中的 Go CI、镜像构建、部署和安全加固。
- 有一个能运行 GitHub Actions 的测试仓库。
- 最好已有 staging 或 Kubernetes/GitOps 部署链路。
- 本机安装 `gh`、`jq`、`docker`、`kubectl` 会更方便。

## 学完后你应该能做到

- 解释 CI/CD 可观测性要看哪些信号。
- 通过 workflow 日志和 summary 快速定位失败。
- 收集并分析 workflow run 和 job 耗时。
- 建立优化前后对比基线。
- 优化 Go cache 和 Docker build cache。
- 使用并行 job、matrix、路径过滤和 concurrency。
- 治理 flaky test 和重复失败。
- 记录部署事件并做发布后验证。
- 给 Go 服务暴露 Prometheus metrics。
- 设计 dashboard、告警和 SLO。
- 用 DORA 指标描述交付效率。
- 写出一份流水线优化报告。

## 推荐官方资料

- GitHub Actions dependency caching：<https://docs.github.com/actions/using-workflows/caching-dependencies-to-speed-up-workflows>
- GitHub Actions concurrency：<https://docs.github.com/actions/using-jobs/using-concurrency>
- GitHub Actions workflow commands：<https://docs.github.com/actions/using-workflows/workflow-commands-for-github-actions>
- GitHub REST API for workflow runs：<https://docs.github.com/rest/actions/workflow-runs>
- Docker build cache：<https://docs.docker.com/build/cache/>
- OpenTelemetry Go：<https://opentelemetry.io/docs/languages/go/>
- Prometheus instrumentation：<https://prometheus.io/docs/practices/instrumentation/>
- Argo CD metrics：<https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/>
- Grafana alerting：<https://grafana.com/docs/grafana/latest/alerting/>
- DORA metrics：<https://dora.dev/guides/dora-metrics-four-keys/>

