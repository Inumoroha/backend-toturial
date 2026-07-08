# 第 3 阶段：文档建模

文档建模是 MongoDB 学习中最容易被低估、也最能区分水平的一部分。CRUD 只是会操作数据库，建模才决定你写出来的系统是否好查、好改、好扩展、好维护。

本阶段的目标不是背“嵌入好”或“引用好”这种简单结论，而是学会从业务访问模式出发，判断数据应该放在同一个文档里、拆成多个集合、做适当冗余、加生命周期字段，还是使用 Schema Validation 约束数据质量。

## 本阶段目标

完成本阶段后，你应该能做到：

- 根据业务查询场景反推集合设计。
- 判断什么时候使用嵌入式模型。
- 判断什么时候使用引用式模型。
- 设计一对一、一对多、多对多关系。
- 识别无限增长数组、超大文档、过度嵌套等风险。
- 合理使用冗余字段，并设计同步策略。
- 为软删除、归档、冷热数据设计生命周期字段。
- 使用 MongoDB Schema Validation 做基础结构约束。
- 完成一个电商订单系统的数据模型设计。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-建模思维-从表设计到访问模式.md` | 从关系型思维转向访问模式思维 |
| 02 | `02-嵌入式模型.md` | 学习嵌入文档的适用场景和风险 |
| 03 | `03-引用式模型.md` | 学习引用关系、两步查询和 `$lookup` 边界 |
| 04 | `04-一对一一对多多对多建模.md` | 设计常见关系模型 |
| 05 | `05-读写模式与反范式设计.md` | 根据读写比例做冗余和快照 |
| 06 | `06-数组增长与文档大小限制.md` | 处理无限增长数组和 16MiB 文档限制 |
| 07 | `07-数据生命周期-软删除归档冷热数据.md` | 设计数据状态、删除、归档和冷热分层 |
| 08 | `08-Schema-Validation结构校验.md` | 使用 JSON Schema 约束集合 |
| 09 | `09-建模模式速查.md` | 常见建模模式和使用建议 |
| 10 | `10-阶段练习-电商订单系统建模.md` | 综合练习：电商订单系统 |
| 11 | `11-阶段验收与建模评审清单.md` | 验收问题和建模评审清单 |

## 本阶段统一思路

建模时按这个顺序思考：

```text
1. 明确业务对象
2. 列出核心查询
3. 判断数据是否一起读取
4. 判断数据是否一起更新
5. 判断数据是否会无限增长
6. 判断数据是否有独立生命周期
7. 决定嵌入、引用或冗余
8. 设计索引和校验规则
9. 写出样例文档
10. 用真实查询验证模型
```

如果你只记住一句话：

> MongoDB 建模不是先画表，而是先画业务访问路径。

## 官方资料

本阶段主要参考：

- Data Modeling：<https://www.mongodb.com/docs/manual/data-modeling/>
- Schema Design Process：<https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/>
- Embedded Data Models：<https://www.mongodb.com/docs/manual/data-modeling/concepts/embedding-vs-references/>
- Schema Validation：<https://www.mongodb.com/docs/manual/core/schema-validation/>
- BSON Document Size：<https://www.mongodb.com/docs/manual/reference/limits/#bson-documents>

