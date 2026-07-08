# 08：测试分层与 Flaky Failure 治理

## 1. 本节目标

CI 最怕两件事：

```text
太慢，开发者不想等。
不可信，开发者不相信失败结果。
```

flaky test 是 CI 信任感的杀手。

这一节学习：

- 如何分层测试。
- 如何分类失败。
- 如何识别 flaky test。
- 如何处理偶发失败而不是掩盖它。

## 2. 测试分层

Go 后端常见测试层级：

```text
unit test：纯函数、业务逻辑、速度快。
integration test：数据库、消息队列、外部组件。
contract test：服务之间契约。
e2e test：完整链路。
smoke test：发布后最小可用验证。
```

推荐 CI 分布：

| 阶段 | 执行内容 |
| --- | --- |
| PR | unit test、lint、轻量安全检查 |
| main | unit + integration、镜像构建、扫描 |
| staging | deploy + smoke test |
| nightly | 慢 e2e、压力或兼容性测试 |
| production | smoke test、监控验证 |

## 3. Go test 常用命令

基础：

```bash
go test ./...
```

race：

```bash
go test -race ./...
```

覆盖率：

```bash
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
```

JSON 输出：

```bash
go test -json ./... > reports/go-test.json
```

只运行短测试：

```bash
go test -short ./...
```

在测试中跳过慢测试：

```go
func TestSlowIntegration(t *testing.T) {
	if testing.Short() {
		t.Skip("skip integration test in short mode")
	}
}
```

## 4. 集成测试标签

可以用 build tags 区分集成测试。

文件头：

```go
//go:build integration

package repository_test
```

运行：

```bash
go test -tags=integration ./...
```

PR 中只跑：

```bash
go test -short ./...
```

main 或 nightly 中跑：

```bash
go test -tags=integration ./...
```

## 5. 失败分类

不要把所有失败都叫“CI 挂了”。

建议分类：

| 类型 | 例子 |
| --- | --- |
| code failure | 单元测试断言失败 |
| environment failure | 数据库容器启动失败 |
| dependency failure | Go proxy、registry、外部 API 不可用 |
| infrastructure failure | runner 掉线、磁盘不足 |
| permission failure | token、OIDC、secret 权限不对 |
| flaky failure | 重跑通过，原因不明确 |

分类的价值：

```text
代码失败找开发者。
环境失败找测试环境负责人。
权限失败找平台/DevOps。
flaky failure 必须进入治理列表。
```

## 6. Flaky test 的信号

可疑特征：

- 同一个 commit 重跑后通过。
- 失败和执行顺序有关。
- 失败和时间有关。
- 失败只在 CI 出现，本地很难复现。
- 错误信息包含 timeout、connection reset、race、port already in use。

常见原因：

- 时间依赖。
- 并发竞态。
- 测试共享全局状态。
- 随机端口冲突。
- 外部服务不稳定。
- 测试数据没有隔离。

## 7. 不要用重试掩盖问题

可以临时重试：

```text
用于降低生产发布被偶发基础设施问题阻断的概率。
```

但不能把重试当修复。

如果使用重试，必须：

- 记录重试次数。
- 在 summary 中标记。
- 给 flaky test 建 issue。
- 限期修复。

否则团队会逐渐默认：

```text
失败了就 rerun。
```

这会让 CI 失去信任。

## 8. 保存测试报告

建议：

```yaml
- name: Test
  run: |
    mkdir -p reports
    go test -json ./... > reports/go-test.json

- name: Upload test report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: go-test-report
    path: reports/go-test.json
    retention-days: 14
```

后续可以用脚本统计：

```text
失败最多的 package。
失败最多的 test case。
平均耗时最长的 test。
重跑后通过的 test。
```

## 9. Flaky 治理流程

建议流程：

```text
1. 记录 flaky 现象。
2. 标记 test name、package、commit、run id。
3. 判断影响范围。
4. 如果阻断大量 PR，可短期隔离。
5. 建 issue 并指定 owner。
6. 修复后恢复到正常 CI。
7. 复盘根因。
```

隔离不是删除。

隔离后的测试要进入：

```text
quarantine workflow
nightly workflow
flaky dashboard
```

## 10. 小练习

完成：

1. 为项目区分 unit、integration、smoke test。
2. 让 PR 默认运行 `go test -short ./...`。
3. 让 main 或 nightly 运行 integration test。
4. 保存 `go test -json` artifact。
5. 写一个 flaky test 处理规则。

## 11. 本节小结

你现在应该理解：

- 测试分层能同时兼顾速度和信心。
- failure 要分类，不能都叫“CI 挂了”。
- flaky test 会破坏团队对 CI 的信任。
- 重试只能缓解，不能替代修复。
- 测试报告和失败历史是治理 flaky 的基础。

