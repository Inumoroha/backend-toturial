# 指标设计模板

## 指标基本信息

```text
指标名：
业务/技术域：
负责人：
首次版本：
```

## 语义

```text
测量对象：
指标类型：
单位：
是否单调：
进程重启后的行为：
采集时机：
统计边界：
```

## Label 契约

| Label | 允许值 | 最大数量 | 来源 | 是否用户输入 |
| --- | --- | ---: | --- | --- |
| method | GET/POST/... | 5 | HTTP | 否 |
| route | 路由模板 | 30 | Router | 否 |
| status | 2xx/4xx/5xx | 8 | Response | 否 |
| result | 固定枚举 | 6 | 业务映射 | 否 |

## 基数和成本

```text
单实例理论 series：
实例数量：
Histogram bucket 数：
预计样本/s：
保留时间：
远程写入影响：
预算和报警：
```

## 查询和告警

```text
主要 Dashboard 查询：
主要 Recording Rule：
主要 Alerting Rule：
SLO 关系：
Runbook：
```

## 禁止字段

```text
user_id
order_id
完整 URL
原始错误文本
请求参数
trace_id
```

## 评审问题

- [ ] 名称符合命名规范。
- [ ] 类型和单位明确。
- [ ] label 值域有限。
- [ ] 没有无界用户输入。
- [ ] 分子/分母和聚合维度可被查询。
- [ ] bucket 与 SLO 边界匹配。
- [ ] 已有测试和 Dashboard 计划。

