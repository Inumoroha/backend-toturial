# 02：Workflow 日志、Summary 与失败排查

## 1. 本节目标

开发者每天最常接触的 CI/CD 可观测工具，就是 workflow 日志。

好的日志能让人快速判断：

```text
失败发生在哪一步？
失败是代码问题、环境问题、依赖问题，还是权限问题？
下一步应该怎么修？
```

这一节学习：

- 如何让日志更清晰。
- 如何使用 workflow commands。
- 如何输出 job summary。
- 如何保存失败证据。
- 如何用 `gh` 排查失败。

## 2. 日志要按阶段分组

GitHub Actions 支持日志分组：

```yaml
- name: Test
  run: |
    echo "::group::Go env"
    go version
    go env
    echo "::endgroup::"

    echo "::group::Run tests"
    go test ./...
    echo "::endgroup::"
```

适合分组的内容：

- 环境信息。
- 依赖下载。
- 测试执行。
- 构建输出。
- 扫描摘要。

不要把所有命令都塞进一个巨大 `run` 步骤。

更好的结构：

```yaml
- name: Download modules
  run: go mod download

- name: Test
  run: go test ./...

- name: Build
  run: go build ./cmd/api
```

这样失败定位更清楚。

## 3. 输出 warning 和 error

可以用 workflow command 输出结构化提示：

```yaml
- name: Check coverage
  run: |
    coverage=68
    if [ "$coverage" -lt 70 ]; then
      echo "::warning title=Low coverage::Coverage is ${coverage}%"
    fi
```

输出错误：

```yaml
- name: Validate config
  run: |
    if [ ! -f config.example.yaml ]; then
      echo "::error title=Missing config::config.example.yaml not found"
      exit 1
    fi
```

注意：

```text
不要在 warning/error 中输出 secret。
```

## 4. 使用 Job Summary

`$GITHUB_STEP_SUMMARY` 可以向 workflow 页面输出 Markdown 摘要。

示例：

```yaml
- name: Write summary
  run: |
    {
      echo "## CI Summary"
      echo ""
      echo "| Item | Value |"
      echo "| --- | --- |"
      echo "| Commit | \`${GITHUB_SHA}\` |"
      echo "| Go version | $(go version) |"
      echo "| Result | Passed |"
    } >> "$GITHUB_STEP_SUMMARY"
```

可以输出：

- 测试数量。
- 覆盖率。
- 镜像 digest。
- 扫描结果。
- 部署环境。
- smoke test 结果。

## 5. 失败时上传 artifact

测试失败时，保存报告：

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
    path: reports/
    retention-days: 7
```

`if: always()` 表示即使前面失败也尝试上传。

适合上传：

- 测试 JSON。
- 覆盖率文件。
- 镜像扫描报告。
- Helm 渲染结果。
- smoke test 日志。

不适合上传：

- `.env`。
- kubeconfig。
- token。
- 私钥。
- 带敏感用户数据的日志。

## 6. 使用 gh 查看失败

查看最近 workflow：

```bash
gh run list --limit 10
```

查看某次运行：

```bash
gh run view <run-id>
```

查看失败日志：

```bash
gh run view <run-id> --log-failed
```

重新运行失败 job：

```bash
gh run rerun <run-id> --failed
```

观察运行中 workflow：

```bash
gh run watch <run-id>
```

这些命令适合在本地快速排查，不必每次打开网页。

## 7. 失败排查流程

建议按顺序看：

```text
1. 是哪个 workflow 失败？
2. 是哪个 job 失败？
3. 是哪个 step 失败？
4. 失败前最后一个关键日志是什么？
5. 是稳定复现还是偶发？
6. 最近是否改过依赖、workflow、Dockerfile、环境变量？
7. 是否只在 main、PR、fork PR 或 schedule 中失败？
```

如果失败和 secret、权限、部署有关，还要看：

```text
permissions 是否足够？
environment 是否需要审批？
secret 是否只在特定 event 可用？
OIDC trust policy 是否匹配 branch/ref？
```

## 8. Debug logging

GitHub Actions 支持启用额外 debug 日志。

常见方式是在 repository secrets 或 variables 中设置：

```text
ACTIONS_STEP_DEBUG=true
ACTIONS_RUNNER_DEBUG=true
```

注意：

- 只在排查时临时开启。
- 排查完成后关闭。
- 不要因为 debug 输出变多而泄露敏感信息。

## 9. 小练习

改造你的 `ci.yml`：

1. 给测试步骤增加日志分组。
2. 失败时上传 `reports/go-test.json`。
3. 在 `$GITHUB_STEP_SUMMARY` 中输出 Go 版本、commit、测试结果。
4. 使用 `gh run view --log-failed` 查看一次失败日志。
5. 写出一次失败的排查记录。

## 10. 本节小结

你现在应该理解：

- 日志结构会直接影响排障效率。
- workflow command 可以输出 warning/error/group。
- job summary 能让结果更容易读。
- artifact 是失败证据，但不能包含 secret。
- `gh run` 是日常排查 GitHub Actions 的利器。

