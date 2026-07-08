# 02 模型设计

本阶段目标：掌握 GORM 模型定义规则，把业务实体设计成清晰、可靠、可维护的数据库结构。

## 学习顺序

1. [模型约定与字段映射](./01-model-conventions.md)
2. [字段标签、索引与软删除](./02-tags-index-soft-delete.md)
3. [博客系统模型设计练习](./03-blog-model-practice.md)
4. [从业务模型到数据库表结构](./04-model-to-sql-practice.md)
5. [主键、时间字段与基础模型](./05-主键时间字段与基础模型.md)
6. [字段类型与 Tag 设计](./06-字段类型与Tag设计.md)
7. [索引与约束建模](./07-索引与约束建模.md)
8. [AutoMigrate 与迁移边界](./08-AutoMigrate与迁移边界.md)
9. [模型设计评审清单](./09-模型设计评审清单.md)

## 本阶段重点

- GORM 默认命名约定。
- `gorm.Model` 的含义。
- 字段 tag 的写法。
- 主键、唯一索引、普通索引。
- `CreatedAt`、`UpdatedAt`、`DeletedAt`。
- 结构体零值与更新行为。

## 本阶段最终产出

你要完成博客系统的基础模型设计：

- User
- Article
- Category
- Tag
- Comment
