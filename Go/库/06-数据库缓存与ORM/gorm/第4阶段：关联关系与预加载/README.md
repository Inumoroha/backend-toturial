# 04 关联关系

本阶段目标：掌握真实项目中最重要的表关系处理，包括一对一、一对多、属于、多对多、预加载和关联操作。

## 学习顺序

1. [关系类型与模型定义](./01-relation-models.md)
2. [预加载与 N+1 查询](./02-preload-and-n-plus-one.md)
3. [关联创建、更新与删除](./03-association-operations.md)
4. [文章、标签、评论关联实战](./04-article-tag-comment-practice.md)
5. [Belongs To 属于关系](./05-BelongsTo属于关系.md)
6. [Has Many 一对多关系](./06-HasMany一对多关系.md)
7. [Many2Many 多对多关系](./07-Many2Many多对多关系.md)
8. [Preload 高级用法](./08-Preload高级用法.md)
9. [关联查询测试与验收](./09-关联查询测试与验收.md)

## 本阶段重点

- `has one`
- `has many`
- `belongs to`
- `many2many`
- `Preload`
- 条件预加载
- 嵌套预加载
- 避免 N+1 查询

## 本阶段最终产出

完善博客系统模型，使它能表达：

- 用户拥有多篇文章。
- 文章属于用户。
- 文章属于分类。
- 文章拥有多条评论。
- 评论属于用户。
- 文章和标签是多对多。
