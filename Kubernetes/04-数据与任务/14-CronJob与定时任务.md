# 14. CronJob 与定时任务

本节目标：学完后你能用 CronJob 执行业务定时任务，例如订单超时关闭和日志清理。

## 简短引入

后端项目里经常有定时任务。以前可能写在 crontab 或某台服务器上。放到 Kubernetes 后，可以用 CronJob 声明任务计划和运行方式。

## 一、为什么需要它

常见业务任务：

- 每分钟关闭超时未支付订单。
- 每天清理过期短链接。
- 每小时汇总访问日志。
- 每晚生成对账文件。

这些任务不能悄悄藏在某台机器的 crontab 里，否则迁移和排障都很困难。

## 二、基本用法

CronJob 示例：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: short-link-expire-cleaner
spec:
  schedule: "*/10 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cleaner
              image: short-api:0.1.0
              imagePullPolicy: IfNotPresent
              command: ["/app/short-api"]
              args: ["jobs", "clean-expired-links"]
              envFrom:
                - secretRef:
                    name: short-api-secret
```

应用和查看：

```bash
kubectl apply -f cronjob.yaml
kubectl get cronjob
kubectl get jobs
```

手动触发一次：

```bash
kubectl create job --from=cronjob/short-link-expire-cleaner short-link-expire-cleaner-manual
kubectl logs job/short-link-expire-cleaner-manual
```

## 三、关键参数/语法/代码结构

- `schedule`：cron 表达式。
- `concurrencyPolicy: Forbid`：上一轮没结束时，不启动下一轮。
- `successfulJobsHistoryLimit`：保留成功 Job 数量。
- `failedJobsHistoryLimit`：保留失败 Job 数量。
- `jobTemplate`：每次触发时创建的 Job 模板。

真实项目中，定时任务要考虑幂等。比如关闭订单时，SQL 条件应包含状态判断：

```sql
UPDATE orders
SET status = 'closed'
WHERE status = 'pending'
  AND expire_at < now();
```

## 四、真实后端场景示例

订单超时关闭任务：

1. 查询 `pending` 且已过期的订单。
2. 开启事务。
3. 修改订单状态。
4. 释放库存。
5. 写操作日志。
6. 提交事务。

```text
定时任务尤其要注意幂等。重复执行一次，结果也应该是可接受的。
```

如果任务可能处理大量数据，不要一次性扫全表。可以分页、分批、限制每轮处理数量。

## 五、注意点

- CronJob 的时间通常按集群控制平面时区理解，生产要明确时区策略。
- `Allow` 并发可能导致同一个任务多实例同时处理同一批数据。
- 任务失败要有告警，不要只保留日志。
- 任务处理数据时要注意事务边界和索引。

## 六、常见错误

- 错误：任务没有幂等设计。
  重试或并发执行时可能重复扣库存、重复发消息。
- 错误：定时任务一次扫全表。
  数据量增长后会拖慢数据库。
- 错误：失败 Job 被自动清理后没人知道。
  生产必须接入日志和告警。

## 七、本节达标标准

- 能创建 CronJob。
- 能手动触发 CronJob 对应的 Job。
- 能解释 `concurrencyPolicy` 的作用。
- 能说明定时任务的幂等和事务边界。

## 八、读懂 Cron 和调度行为

五段 Cron 表达式从左到右是：分钟、小时、日、月、星期。

```text
*/10 * * * *   每 10 分钟
0 2 * * *      每天 02:00
30 9 * * 1-5   周一到周五 09:30
```

生产中必须明确时区。集群支持 `spec.timeZone` 时可显式设置，例如 `Asia/Shanghai`；提交前用 `kubectl explain cronjob.spec.timeZone` 检查当前版本是否支持。

不要等待十分钟验证任务，直接从 CronJob 创建一次性 Job：

```bash
kubectl create job --from=cronjob/short-link-expire-cleaner cleaner-check-001
kubectl get job cleaner-check-001 -w
kubectl logs job/cleaner-check-001
```

自测：为什么 `concurrencyPolicy: Forbid` 仍不能代替幂等设计？因为任务可能在写入成功后、报告完成前失败，随后被重试；外部系统也可能重复触发。
