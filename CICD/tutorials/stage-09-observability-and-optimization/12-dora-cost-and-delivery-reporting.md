# 12：DORA、成本与交付报告

## 1. 本节目标

到这里，你已经能观察单次 workflow 和单次发布。

这一节把视角提升到团队交付效率：

- DORA 四项指标。
- runner 和 artifact 成本。
- 交付周报。
- 如何避免指标被滥用。

## 2. DORA 四项指标

DORA 常用四项：

```text
Deployment Frequency：部署频率。
Lead Time for Changes：变更前置时间。
Change Failure Rate：变更失败率。
Time to Restore Service：服务恢复时间。
```

它们分别回答：

| 指标 | 问题 |
| --- | --- |
| 部署频率 | 我们多久交付一次？ |
| 变更前置时间 | 从提交到上线要多久？ |
| 变更失败率 | 上线后出问题的比例是多少？ |
| 恢复时间 | 出问题后多久恢复？ |

## 3. 如何定义部署频率

先明确什么算一次部署。

建议：

```text
production 环境成功发布一次算一次部署。
staging 不计入核心 DORA 部署频率，但可以单独观察。
```

数据来源：

- GitHub deployment。
- 部署仓库 merge commit。
- Argo CD history。
- release workflow run。

## 4. 如何定义变更前置时间

常见定义：

```text
从 commit 合并到 main，到该 commit 部署到 production 的时间。
```

也可以使用：

```text
从 PR 创建到 production 发布。
```

两者含义不同。

初学阶段建议用：

```text
main merge time -> production deploy complete time
```

因为它更容易从 Git 和部署记录中获取。

## 5. 如何定义变更失败率

一次 production 发布后，如果导致：

- 回滚。
- hotfix。
- 高优先级事故。
- 用户可见故障。
- 错误预算明显燃烧。

就可以标记为 failed change。

公式：

```text
change failure rate = failed production deployments / total production deployments
```

注意：

```text
不要把所有小 bug 都算成变更失败。
要先定义清楚团队标准。
```

## 6. 如何定义恢复时间

恢复时间通常是：

```text
从故障开始或被发现，到服务恢复到可接受状态。
```

数据来源：

- 告警触发时间。
- incident timeline。
- 回滚完成时间。
- SLO 恢复时间。

关键是保持一致。

## 7. 不要把 DORA 变成个人考核武器

DORA 指标应该帮助团队发现系统问题：

```text
审批流程是否太慢？
测试是否不稳定？
发布是否太痛苦？
回滚是否困难？
```

不应用来简单评价个人：

```text
你这个月部署次数太少。
你导致失败率上升。
```

错误使用指标会让团队开始规避记录，而不是改善系统。

## 8. Runner 成本治理

成本来源：

- GitHub-hosted runner minutes。
- 自托管 runner 机器成本。
- 大规格 runner。
- 重复运行。
- 过长 artifact retention。
- 过大的缓存。
- 不必要的 matrix。

优化方向：

- PR 取消过时 run。
- 文档变更跳过重型 workflow。
- cache 命中率提升。
- slow test 分层。
- artifact 设置合理保留天数。
- release 才做多平台构建。

查看 GitHub Actions cache：

```bash
gh cache list
```

删除 cache：

```bash
gh cache delete <cache-id>
```

## 9. Artifact 保留策略

示例：

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: test-report
    path: reports/
    retention-days: 14
```

建议：

| Artifact | 保留 |
| --- | --- |
| PR test report | 7 到 14 天 |
| release SBOM | 更长，按合规要求 |
| scan report | 30 到 90 天 |
| debug dump | 排障结束后删除 |

不要长期保存含敏感信息的 artifact。

## 10. 交付周报模板

创建：

```text
docs/delivery-report-template.md
```

内容：

```markdown
# Delivery Report

Period: YYYY-MM-DD to YYYY-MM-DD

## CI Health

| Metric | Value | Trend |
| --- | --- | --- |
| PR CI p50 | | |
| PR CI p95 | | |
| CI success rate | | |
| Most failed job | | |

## Deployment

| Metric | Value | Trend |
| --- | --- | --- |
| Production deployments | | |
| Change failure rate | | |
| Mean restore time | | |

## Cost

| Metric | Value | Trend |
| --- | --- | --- |
| Runner minutes | | |
| Artifact storage | | |
| Cache size | | |

## Improvements

- Done:
- Next:
- Risks:
```

## 11. 小练习

完成：

1. 定义你项目中的一次 production deployment。
2. 定义 change failure。
3. 统计最近 1 周 production 发布次数。
4. 查看最近 CI p50/p95。
5. 设计 artifact retention 策略。
6. 写一份简短交付周报。

## 12. 本节小结

你现在应该理解：

- DORA 指标描述团队交付系统，不是个人绩效标签。
- 每个指标都要先定义清楚。
- runner、cache、artifact 都有成本。
- 周报应该展示趋势、问题和下一步行动。
- 成本优化不能牺牲必要的质量和安全门禁。

