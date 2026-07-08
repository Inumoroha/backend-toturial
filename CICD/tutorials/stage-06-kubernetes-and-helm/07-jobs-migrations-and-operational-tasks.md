# 07：Job、CronJob 与迁移任务

## 1. 本节目标

第 5 阶段我们讲过数据库迁移风险。

在 Kubernetes 中，一次性任务更适合用 Job 表达。

这一节学习：

- Job。
- CronJob。
- 迁移 Job。
- 运维任务和应用 Deployment 的边界。

## 2. Job 是什么

Job 用于运行一次性任务，直到成功完成。

适合：

- 数据库迁移。
- 一次性数据修复。
- 初始化任务。
- 批处理。

Job 和 Deployment 不同：

- Deployment 期望长期运行。
- Job 期望运行完成后退出。

## 3. 迁移 Job 示例

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: go-cicd-lab-migrate
  namespace: go-cicd-lab
spec:
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: go-cicd-lab:local
          imagePullPolicy: IfNotPresent
          command: ["/server"]
          args: ["migrate", "up"]
          envFrom:
            - configMapRef:
                name: go-cicd-lab-config
            - secretRef:
                name: go-cicd-lab-secret
```

前提：你的 Go 程序支持：

```bash
/server migrate up
```

## 4. 运行 Job

```bash
kubectl apply -f job-migrate.yaml
kubectl -n go-cicd-lab wait --for=condition=complete job/go-cicd-lab-migrate --timeout=120s
kubectl -n go-cicd-lab logs job/go-cicd-lab-migrate
```

删除 Job：

```bash
kubectl -n go-cicd-lab delete job go-cicd-lab-migrate
```

如果 Job 名字固定，重复 apply 可能不会重新执行。常见做法是让 CI/Helm 用唯一名字，或先删除旧 Job。

## 5. 迁移和 Helm hook

Helm 支持 hooks，例如 pre-upgrade Job。

但初学阶段先谨慎：

- hook 失败会影响 release。
- hook 资源生命周期要管理。
- 数据库迁移不可随便自动回滚。

建议先手动或 CI 显式运行迁移 Job，理解流程后再使用 hook。

## 6. CronJob

CronJob 用于定时任务。

示例：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: go-cicd-lab-cleanup
  namespace: go-cicd-lab
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: go-cicd-lab:local
              command: ["/server"]
              args: ["cleanup"]
```

适合：

- 定时清理。
- 定时同步。
- 定时统计。

不适合高精度调度或复杂工作流。

## 7. 迁移安全建议

生产迁移要有：

- migration 文件审查。
- 备份或恢复方案。
- 兼容性说明。
- 执行窗口。
- 回滚或补偿方案。

不要把高风险破坏性迁移隐藏在自动部署里。

## 8. 小练习

1. 写一个 Job YAML，运行 `/server version` 或类似命令。
2. apply Job。
3. 查看 Job 状态。
4. 查看 logs。
5. 删除 Job。
6. 思考迁移 Job 应该在 Deployment 更新前还是后执行。

## 9. 本节小结

你现在应该理解：

- Job 适合一次性任务，Deployment 适合长期服务。
- 数据库迁移更适合显式 Job。
- Helm hook 有用，但要谨慎。
- CronJob 用于定时任务。

