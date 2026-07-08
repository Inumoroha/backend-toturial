# 03 查询进阶

本阶段目标：熟练使用 GORM 完成真实业务中常见的查询：条件、排序、分页、统计、聚合、原生 SQL 和结果扫描。

## 学习顺序

1. [条件查询、排序与分页](./01-where-order-pagination.md)
2. [统计、分组与聚合查询](./02-count-group-aggregate.md)
3. [原生 SQL 与结果扫描](./03-raw-sql-scan.md)
4. [博客查询 Repository 实战](./04-blog-query-repository-practice.md)
5. [动态查询条件模式](./05-动态查询条件模式.md)
6. [分页查询模式](./06-分页查询模式.md)
7. [Joins 与 Preload 查询取舍](./07-Joins与Preload查询取舍.md)
8. [搜索、筛选与安全排序](./08-搜索筛选与安全排序.md)
9. [查询测试与 SQL 验证](./09-查询测试与SQL验证.md)

## 本阶段重点

- `Where` 的安全写法。
- `Select`、`Order`、`Limit`、`Offset`。
- `Count`、`Group`、`Having`。
- `Raw`、`Exec`、`Scan`。
- 查询结果为空时如何处理。
- 什么时候该用 GORM API，什么时候该写原生 SQL。

## 本阶段最终产出

基于博客系统完成：

- 文章列表接口查询逻辑。
- 文章详情查询逻辑。
- 按关键词搜索文章。
- 分类文章数量统计。
- 热门文章查询。
