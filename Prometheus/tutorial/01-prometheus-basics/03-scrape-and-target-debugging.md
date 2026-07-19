# Target、抓取与排障

## 1. Target 页面应该看什么

打开 `http://localhost:9090/targets`，逐个观察：

- State：`UP`、`DOWN` 或其他状态。
- Last Scrape：最近一次抓取时间。
- Scrape Duration：抓取耗时。
- Error：最近一次失败的具体原因。
- Discovered labels 和 labels：服务发现与最终标签是否符合预期。

不要只看 `up`。target 页面提供了比一条查询更多的错误上下文。

## 2. 通过 API 查看 target

```bash
curl -s http://localhost:9090/api/v1/targets | jq .
```

没有 jq 时，先保存 JSON，再使用编辑器查看：

```bash
curl -s http://localhost:9090/api/v1/targets > targets.json
```

查询 PromQL：

```bash
curl -G http://localhost:9090/api/v1/query \
  --data-urlencode 'query=up'
```

## 3. 四种故障实验

### 实验 A：进程不可达

```bash
docker compose stop app
```

预期：

- target 变为 DOWN。
- `up{job="demo-app"}` 变成 0。
- Error 通常包含 connection refused 或 timeout。

恢复：

```bash
docker compose start app
```

### 实验 B：路径错误

把 `metrics_path` 改为 `/metric` 并 reload。

预期：

- TCP 可连接。
- HTTP 返回 404。
- target 仍然 DOWN。

### 实验 C：格式错误

使用普通文本服务作为 target。

预期：

- HTTP 请求可能成功。
- Prometheus 解析失败。
- 不能因为“curl 有返回”就认为抓取成功。

### 实验 D：抓取超时

在应用 `/metrics` 处理器中加入大于 `scrape_timeout` 的延迟。

预期：

- target 错误是 context deadline exceeded 或 timeout。
- `scrape_duration_seconds` 接近超时上限。
- 需要先修复服务端响应或调整超时边界，而不是无条件延长超时。

## 4. scrape 配置中的重要字段

```yaml
scrape_configs:
  - job_name: go-app
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets: ["app:8080"]
        labels:
          environment: local
```

原则：

- `scrape_timeout` 必须小于 `scrape_interval`。
- metrics handler 要轻量、无外部依赖或有严格超时。
- `job` 和 `instance` 是排障时最重要的默认维度。
- 静态 labels 用于环境、团队等稳定上下文，不要写请求级数据。

## 5. relabel 的第一印象

target relabel 发生在抓取前，适合：

- 丢弃不需要的 target。
- 改名或补充 target label。
- 从服务发现元数据生成最终 label。

metric relabel 发生在样本抓取后，适合：

- 丢弃特定指标。
- 删除不需要的样本 label。

metric relabel 会消耗抓取、解析和内存成本，不能把它当作高基数埋点的替代修复方案。

## 6. 诊断检查表

| 检查 | 命令或页面 | 证据 |
| --- | --- | --- |
| Prometheus 就绪 | `/-/ready` | HTTP 200 |
| target 状态 | `/targets` | State 和 Last Error |
| 网络连通 | 容器内 curl/wget | 连接和响应 |
| 路径 | `/metrics` | HTTP 状态 |
| 格式 | `promtool check metrics` | 解析结果 |
| 时间序列 | `up`、`scrape_samples_scraped` | 查询结果 |
| 规则加载 | `/rules` | 评估状态 |

## 7. 阶段验收

你需要提交一份故障记录，至少包含：

- 故障开始和恢复时间。
- target 页面截图或 API 输出。
- 一条能复现的命令。
- 根因所在的层次。
- 修复后如何验证没有残留问题。

