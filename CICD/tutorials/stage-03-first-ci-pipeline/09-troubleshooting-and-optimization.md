# 09：CI 排障与优化

## 1. 本节目标

CI 不是写完 YAML 就结束。

真实工作里，你会经常处理：

- CI 本地不能复现。
- 缓存失效。
- 依赖下载失败。
- 测试偶发失败。
- runner 环境差异。
- job 太慢。
- required check 名称变化。

这一节学习排障方法。

## 2. 先定位失败层级

看到红叉时，先问：

```text
是 workflow 没触发？
是 job 没启动？
是 setup 环境失败？
是依赖失败？
是命令失败？
是测试失败？
是权限失败？
```

不要直接猜。

按照页面进入：

```text
Actions
-> workflow run
-> failed job
-> failed step
-> log lines
```

## 3. 常见问题：workflow 没触发

检查：

- 文件是否在 `.github/workflows/` 下。
- 文件扩展名是否是 `.yml` 或 `.yaml`。
- YAML 是否语法错误。
- `on` 触发条件是否匹配。
- PR 目标分支是不是 main。
- workflow 文件是否已提交到对应分支。

示例问题：

```yaml
on:
  pull_request:
    branches: [master]
```

但你的主分支叫 main，这就不会按预期触发。

## 4. 常见问题：Go 版本不一致

现象：

```text
本地测试通过，CI 编译失败。
```

检查：

```yaml
with:
  go-version: "1.25.x"
```

以及：

```text
go.mod 中的 go 版本
```

建议：

- 本地和 CI 使用相同主版本。
- 可以使用 `go-version-file: go.mod`。
- CI 中打印 `go version`。

## 5. 常见问题：go.sum 没提交

现象：

```text
go mod download 失败
go test 提示 go.sum entry missing
```

解决：

```bash
go mod tidy
git add go.mod go.sum
git commit -m "chore: tidy go modules"
```

## 6. 常见问题：Makefile 在 CI 失败

常见原因：

- 本地 Windows 命令写进了 Makefile。
- CI 是 Linux runner。
- 缺少 `make`。
- 缺少工具。
- 脚本没有执行权限。

GitHub Ubuntu runner 通常有 `make`。

如果是自建 runner 或特殊镜像，要自己确认。

## 7. 常见问题：golangci-lint 本地和 CI 结果不同

原因可能是：

- 版本不同。
- Go 版本不同。
- 配置文件没有提交。
- CI 使用了默认配置。

解决：

- 固定 golangci-lint 版本。
- 提交 `.golangci.yml`。
- 在文档中写明本地版本。

## 8. 常见问题：测试偶发失败

偶发失败比稳定失败更危险。

常见原因：

- 测试依赖时间。
- 测试依赖执行顺序。
- 测试共享全局变量。
- 测试使用固定端口。
- 并发测试没有隔离数据。
- 外部网络不稳定。

处理：

- 避免真实 sleep。
- 使用 fake clock。
- 测试数据隔离。
- 避免依赖测试顺序。
- 不让单元测试访问外网。
- 对并发逻辑跑 race。

## 9. 常见问题：CI 太慢

先看耗时分布，不要盲目优化。

优化顺序：

1. 启用 Go cache。
2. 拆并行 job。
3. 避免重复安装工具。
4. 把特别慢的检查放到 main 或 schedule。
5. 使用 path filter，但要谨慎。
6. 优化慢测试。

不要为了快而删掉关键测试。

## 10. 常见问题：required check 找不到

如果你配置了 required checks，但 PR 一直等待：

检查：

- workflow name 是否改了。
- job name 是否改了。
- 触发条件是否让该 job 没运行。
- 分支保护中选择的 check 名是否已经过期。

建议：

```text
CI 稳定后，不要频繁改 workflow name 和 job name。
```

## 11. 本地复现 CI

CI 失败后，尽量本地复现：

```bash
go mod download
make ci-core
make lint
make vuln
```

如果本地复现不了，比较：

- Go 版本。
- 操作系统。
- 环境变量。
- 是否有缓存。
- 是否依赖未提交文件。
- 是否依赖本地服务。

## 12. Debug 技巧

可以临时加入：

```yaml
- name: Debug Go env
  run: go env
```

可以打印文件结构：

```yaml
- name: List files
  run: find . -maxdepth 3 -type f | sort
```

不要打印 secret 或完整环境变量。

调试完要删除无用 step，保持 CI 干净。

## 13. 小练习

故意制造并修复三个问题：

1. 改错 Go 版本，观察失败。
2. 删除 `go.sum` 中某些内容，观察失败。
3. 写一个失败测试，观察 PR 状态。

然后写下：

- 失败出现在哪个 job。
- 失败出现在哪个 step。
- 关键日志是哪一行。
- 本地如何复现。
- 最终如何修复。

## 14. 本节小结

你现在应该理解：

- CI 排障要先定位 workflow、job、step、命令。
- 本地和 CI 不一致时优先比较环境。
- 偶发失败要认真处理。
- 优化 CI 之前先看耗时。
- required checks 名称要保持稳定。

