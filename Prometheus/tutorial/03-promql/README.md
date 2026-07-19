# 阶段 3：PromQL

PromQL 的目标是把样本转成决策：流量是否增加、错误是否影响用户、延迟是否违反 SLO、哪个实例最异常。学习时始终围绕问题写查询。

## 学习目标

- 理解 instant vector、range vector、scalar 和空结果。
- 熟练使用 selector、聚合、`rate`、`increase` 和时间函数。
- 用 Histogram 计算可聚合的分位数。
- 理解 label matching，避免 join 后序列爆炸。
- 写 Recording Rule，降低重复查询成本。
- 处理 Counter reset、缺失数据和低流量场景。

## 教程顺序

1. [选择器、时间范围与基础运算](01-selectors-time-functions.md)
2. [Counter、速率、聚合与错误率](02-rate-aggregation-error-rate.md)
3. [Histogram 与分位数](03-histogram-quantiles-slo.md)
4. [向量匹配、Recording Rule 与查询优化](04-vector-matching-recording-rules.md)

## 建议查询笔记格式

```text
问题：
数据源：
PromQL：
预期结果：
实际结果：
label 维度：
空结果/异常值的解释：
是否适合做 Recording Rule：
```

## 查询验证顺序

1. 先查询原始指标是否存在。
2. 再加一个 selector。
3. 再观察时间窗口。
4. 再做 `rate` 或时间函数。
5. 再做聚合。
6. 最后才做除法、join 和分位数。

不要一上来写很长的表达式。每一步都有结果，才能知道错误出在哪一层。

## 阶段验收

至少完成路线图中的 30 条查询，并为每条查询写出单位、保留的 label 和空结果含义。

