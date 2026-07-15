# 13. Job 与数据库迁移

本节目标：学完后你能用 Kubernetes Job 执行一次性数据库迁移任务。

## 简短引入

后端项目经常要变更表结构。数据库迁移不应该靠开发手工登录数据库执行 SQL。Kubernetes 中可以用 Job 跑一次性任务，例如迁移、数据修复、离线导入。

## 一、为什么需要它

真实发布中，代码和数据库结构要匹配：

- 新增短链接表字段。
- 给订单表加索引。
- 修复历史用户状态。
- 初始化权限菜单。

这些操作要可观察、可失败、可重试。Job 适合表达“运行一次直到成功或失败”的任务。

```text
数据库迁移要有明确版本、执行记录和回滚思路，不能靠复制 SQL 到生产库临时执行。
```

## 二、基本用法

假设镜像里有迁移命令：

```bash
./short-api migrate up
```

Job 示例：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: short-api-migrate-20260706
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: short-api:0.1.0
          imagePullPolicy: IfNotPresent
          command: ["/app/short-api"]
          args: ["migrate", "up"]
          envFrom:
            - secretRef:
                name: short-api-secret
```

执行和查看：

```bash
kubectl apply -f migrate-job.yaml
kubectl get jobs
kubectl get pods -l job-name=short-api-migrate-20260706
kubectl logs job/short-api-migrate-20260706
```

## 三、关键参数/语法/代码结构

- `restartPolicy: Never`：任务失败后不在同一个 Pod 内无限重启。
- `backoffLimit`：失败重试次数。
- `command` 和 `args`：覆盖镜像默认启动命令。
- Job 名称建议带版本或日期，方便追踪。

Go 项目里可以用成熟迁移工具，例如 golang-migrate、goose、Atlas。核心原则是迁移文件进入版本管理，发布时按版本执行。

## 四、真实后端场景示例

短链接表迁移：

```sql
CREATE TABLE IF NOT EXISTS short_links (
    id BIGSERIAL PRIMARY KEY,
    code VARCHAR(32) NOT NULL UNIQUE,
    original_url TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_short_links_created_at
ON short_links (created_at);
```

这个 SQL 看起来简单，但真实项目要考虑：

- `code` 唯一约束防止短码冲突。
- 索引会提升查询，但会增加写入成本。
- 大表加索引可能锁表或拖慢数据库，需要评估执行方式。

## 五、注意点

- 迁移 Job 不应该和 API Deployment 同时无序执行，CI/CD 要安排顺序。
- 迁移失败时不要继续发布依赖新字段的代码。
- 大表 DDL 要评估锁和执行时间。
- 回滚不一定是简单 drop column，删除数据类迁移尤其要谨慎。

```text
上线顺序通常是：兼容性迁移 -> 发布新代码 -> 清理旧字段。不要让代码和表结构互相卡死。
```

## 六、常见错误

- 错误：每次 apply 都用同一个 Job 名。
  已完成 Job 不会像 Deployment 那样重新执行，通常要新 Job 名或先清理旧 Job。
- 错误：迁移逻辑写在 API 启动流程里。
  多副本同时启动可能并发执行迁移。
- 错误：没有迁移记录表。
  不知道哪些 SQL 已经执行过，回滚和排障困难。

## 七、本节达标标准

- 能创建一个迁移 Job。
- 能查看 Job 日志和执行结果。
- 能说明迁移任务为什么不应放在 API 多副本启动流程中。
- 能理解迁移顺序和回滚风险。

## 八、把 Job 当成有结果的一次执行

Deployment 期望进程长期运行，Job 期望任务成功结束。程序以退出码 0 结束才算成功，非 0 会按策略重试。

```bash
kubectl apply -f migrate-job.yaml
kubectl get job -w
kubectl get pods -l job-name=short-api-migrate-001
kubectl logs job/short-api-migrate-001
kubectl describe job short-api-migrate-001
```

成功时 `COMPLETIONS` 变为 `1/1`。失败时先保留 Job 和 Pod 查日志，不要马上删除现场。

Job 名称需要区分迁移版本，因为已完成的同名 Job 再次 `apply` 不会重新执行任务。可以使用 `short-api-migrate-002`，或由流水线为每次迁移生成可追踪名称。

练习：让迁移命令暂时返回非零退出码，观察重试次数和 `backoffLimit`；恢复命令后创建新名称的 Job。思考任务重复执行时是否安全，这就是幂等性要求。
