# 03：流水线指标与 API 采集

## 1. 本节目标

日志适合排查单次失败。

指标适合回答趋势问题：

```text
最近 CI 有没有变慢？
哪个 workflow 最常失败？
PR 等待反馈的时间是多少？
优化后是否真的变快？
```

这一节学习如何用 GitHub CLI 和 API 收集流水线指标。

## 2. 最小指标集

建议先采集：

| 指标 | 含义 |
| --- | --- |
| run duration | workflow 总耗时 |
| job duration | job 耗时 |
| conclusion | success/failure/cancelled |
| event | pull_request/push/schedule/workflow_dispatch |
| branch | 分支 |
| run attempt | 第几次重跑 |
| created_at | 创建时间 |
| updated_at | 结束更新时间 |

后续再扩展：

- cache hit rate。
- artifact 大小。
- runner 类型。
- deploy duration。
- smoke test result。

## 3. 使用 gh 查看 workflow run

查看最近运行：

```bash
gh run list --limit 20
```

查看指定 workflow：

```bash
gh run list --workflow ci.yml --limit 20
```

输出 JSON：

```bash
gh run list \
  --workflow ci.yml \
  --limit 20 \
  --json databaseId,displayTitle,event,headBranch,conclusion,createdAt,updatedAt
```

如果安装了 `jq`，可以整理成表格：

```bash
gh run list \
  --workflow ci.yml \
  --limit 20 \
  --json databaseId,event,headBranch,conclusion,createdAt,updatedAt \
  --jq '.[] | [.databaseId, .event, .headBranch, .conclusion, .createdAt, .updatedAt] | @tsv'
```

## 4. 使用 GitHub REST API

列出 workflow runs：

```bash
gh api \
  repos/:owner/:repo/actions/runs \
  -f per_page=100 \
  --jq '.workflow_runs[] | [.id, .name, .event, .head_branch, .conclusion, .created_at, .updated_at] | @tsv'
```

查看某个 run 的 jobs：

```bash
gh api \
  repos/:owner/:repo/actions/runs/<run-id>/jobs \
  --jq '.jobs[] | [.name, .conclusion, .started_at, .completed_at] | @tsv'
```

保存 JSON：

```bash
mkdir -p reports

gh api repos/:owner/:repo/actions/runs \
  -f per_page=100 \
  > reports/workflow-runs.json
```

## 5. 计算耗时

用 `jq` 输出原始时间：

```bash
jq -r '
  .workflow_runs[]
  | [.id, .name, .conclusion, .created_at, .updated_at]
  | @tsv
' reports/workflow-runs.json
```

如果想计算秒数，可以用一段小脚本。

创建：

```text
scripts/cicd-run-durations.py
```

内容：

```python
import json
from datetime import datetime, timezone

def parse_time(value):
    return datetime.fromisoformat(value.replace("Z", "+00:00"))

with open("reports/workflow-runs.json", "r", encoding="utf-8") as f:
    data = json.load(f)

for run in data["workflow_runs"]:
    created = parse_time(run["created_at"])
    updated = parse_time(run["updated_at"])
    duration = int((updated - created).total_seconds())
    print(f'{run["id"]}\t{run["name"]}\t{run["conclusion"]}\t{duration}')
```

运行：

```bash
python scripts/cicd-run-durations.py
```

## 6. 计算成功率

简单统计：

```bash
jq -r '.workflow_runs[].conclusion' reports/workflow-runs.json | sort | uniq -c
```

你可能看到：

```text
  82 success
  11 failure
   7 cancelled
```

成功率：

```text
success / (success + failure)
```

是否把 `cancelled` 算进去，取决于你的团队定义。

如果很多 run 是被 `concurrency cancel-in-progress` 取消，它不应该被等同于失败。

## 7. Job 级指标更有价值

workflow 总耗时只能告诉你“慢”。

job 级指标能告诉你“哪里慢”：

```bash
gh api repos/:owner/:repo/actions/runs/<run-id>/jobs \
  > reports/jobs-<run-id>.json
```

查看每个 job：

```bash
jq -r '
  .jobs[]
  | [.name, .conclusion, .started_at, .completed_at]
  | @tsv
' reports/jobs-<run-id>.json
```

建议记录：

```text
setup-go duration
go test duration
Docker build duration
Trivy scan duration
deploy duration
```

如果一个 job 内部有多个 step，就需要结合 workflow 日志或 step duration 分析。

## 8. 指标保存到哪里

学习阶段可以保存到：

```text
reports/
docs/pipeline-optimization-report.md
```

团队阶段可以保存到：

- Prometheus。
- InfluxDB。
- BigQuery。
- ClickHouse。
- Grafana dashboard。
- GitHub issue/comment。
- 内部数据平台。

工具不重要，重要的是稳定地回答：

```text
最近 30 天 PR CI p50/p95 是多少？
失败最多的是哪个 job？
优化后是否改善？
```

## 9. 小练习

完成：

1. 使用 `gh run list --workflow ci.yml --limit 20` 查看最近 CI。
2. 导出最近 100 次 workflow runs。
3. 计算 success/failure/cancelled 数量。
4. 选 3 次最慢的 run，查看 job 耗时。
5. 在 `docs/pipeline-optimization-report.md` 中写下基线。

## 10. 本节小结

你现在应该理解：

- 单次失败看日志，长期趋势看指标。
- workflow run 指标适合看整体趋势。
- job 指标适合定位瓶颈。
- cancelled 是否算失败要有明确团队定义。
- 优化前必须保存可对比的基线。

