# 08：安全与依赖检查

## 1. 本节目标

Go 项目的质量门禁不仅是测试和 lint，还包括安全检查。

这一节学习：

- `govulncheck`。
- 依赖升级。
- secret 防护。
- `.gitignore`。
- 基础供应链意识。

## 2. govulncheck 是什么

`govulncheck` 是 Go 官方漏洞检查工具。

它会分析你的代码和依赖，报告会影响项目的已知漏洞。

安装：

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
```

运行：

```bash
govulncheck ./...
```

如果命令找不到，确认 Go bin 目录在 PATH 中。

查看 Go bin：

```bash
go env GOPATH
```

通常二进制位于：

```text
$GOPATH/bin
```

Windows 可能是：

```text
%USERPROFILE%\go\bin
```

## 3. 如何理解 govulncheck 结果

漏洞检查结果通常会告诉你：

- 漏洞编号。
- 受影响模块。
- 受影响符号。
- 你的代码是否可能调用到漏洞路径。
- 修复版本。

处理顺序：

1. 判断是否可达。
2. 查看修复版本。
3. 升级依赖。
4. 跑测试。
5. 如果不能立即升级，记录风险和缓解措施。

## 4. 升级依赖

查看可升级依赖：

```bash
go list -u -m all
```

升级某个依赖：

```bash
go get example.com/module@latest
go mod tidy
go test ./...
```

升级所有 patch/minor 需要谨慎，真实项目里不要盲目全量升级。

## 5. go mod tidy 也是安全习惯

不再使用的依赖应该移除。

执行：

```bash
go mod tidy
```

然后检查：

```bash
git diff go.mod go.sum
```

如果出现大量变化，要理解原因。

## 6. Secret 防护

不要提交：

```text
.env
.env.local
id_rsa
*.pem
*.key
kubeconfig
service-account.json
```

`.gitignore` 示例：

```text
bin/
coverage.out
.env
.env.*
!.env.example
*.pem
*.key
```

可以提交：

```text
.env.example
```

内容：

```text
APP_ENV=local
HTTP_ADDR=:8080
DATABASE_URL=
```

## 7. 已经提交了 secret 怎么办

如果误提交密钥：

1. 立即撤销或轮换该密钥。
2. 删除代码中的密钥。
3. 清理 Git 历史需要团队谨慎操作。
4. 通知相关人员。
5. 检查访问日志。

不要以为删除最新 commit 就一定安全。只要推送到远程，就要按泄漏处理。

## 8. 安全检查放在哪里

建议：

| 检查 | PR | main | schedule |
| --- | --- | --- | --- |
| `govulncheck ./...` | 可选 | 推荐 | 推荐 |
| secret scan | 推荐 | 推荐 | 推荐 |
| dependency update | 否 | 否 | 推荐 |

学习项目可以先在本地和 main 流水线中跑 `govulncheck`。

第 3 阶段接 CI 时，再考虑定时扫描。

## 9. 供应链安全基础意识

Go 项目依赖第三方模块。CI/CD 还会构建镜像、推送制品、部署环境。

所以你要逐步建立这些意识：

- 依赖版本要明确。
- 构建产物要可追溯。
- 密钥要最小权限。
- CI 日志不能泄漏敏感信息。
- 生产镜像不能包含开发密钥。
- 依赖和镜像要定期扫描。

本阶段只先打基础。后面第 8 阶段会系统学习 CI/CD 安全。

## 10. 小练习

执行：

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
go list -u -m all
go mod tidy
go test ./...
```

然后完成：

1. 新增 `.env.example`。
2. 确保 `.env` 被 `.gitignore` 忽略。
3. 写一段文档说明项目有哪些 secret。

## 11. 本节小结

你现在应该理解：

- `govulncheck` 是 Go 官方漏洞检查工具。
- 依赖升级后必须跑测试。
- secret 不能进入 Git、日志和镜像。
- `.env.example` 可以提交，真实 `.env` 不应该提交。
- 安全检查是质量门禁的一部分。

