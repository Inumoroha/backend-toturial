# 03 关联创建、更新与删除

本节目标：围绕「association operations」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

关联不只用于查询，也会用于创建、替换、追加和删除关联关系。尤其是多对多标签场景，关联操作非常常见。

## 1. 创建文章时指定外键

最简单、最清晰的方式是直接写外键：

```go
article := Article{
    UserID:     userID,
    CategoryID: categoryID,
    Title:      "GORM 入门",
    Content:    "正文内容",
    Status:     "published",
}

err := db.Create(&article).Error
```

真实项目里，创建文章时通常就是这样写。

## 2. 创建文章并关联标签

如果标签已经存在，可以先查询标签，再创建文章：

```go
var tags []Tag
if err := db.Where("id IN ?", tagIDs).Find(&tags).Error; err != nil {
    return err
}

article := Article{
    UserID:     userID,
    CategoryID: categoryID,
    Title:      "GORM 关联",
    Content:    "正文内容",
    Tags:       tags,
}

err := db.Create(&article).Error
```

GORM 会创建文章，并写入中间表 `article_tags`。

## 3. Append 追加关联

给文章追加标签：

```go
var article Article
if err := db.First(&article, articleID).Error; err != nil {
    return err
}

var tags []Tag
if err := db.Where("id IN ?", tagIDs).Find(&tags).Error; err != nil {
    return err
}

err := db.Model(&article).Association("Tags").Append(&tags)
```

`Append` 不会清空原有关联。

## 4. Replace 替换关联

编辑文章时，标签通常是整体替换：

```go
err := db.Model(&article).Association("Tags").Replace(&tags)
```

`Replace` 会把旧标签关系替换成新标签关系。

## 5. Delete 删除指定关联

```go
err := db.Model(&article).Association("Tags").Delete(&tag)
```

这通常只删除中间表关系，不删除标签本身。

## 6. Clear 清空关联

```go
err := db.Model(&article).Association("Tags").Clear()
```

会清空文章的所有标签关系。

## 7. Count 统计关联数量

```go
count := db.Model(&article).Association("Tags").Count()
```

注意：`Association` API 的错误处理方式和普通链式 API 不完全一样，使用时要看返回值。

## 8. 关联操作与事务

编辑文章时，常见步骤：

```text
1. 更新文章标题和正文
2. 替换文章标签
```

这两个操作应该放进事务。否则可能出现文章内容更新成功，但标签更新失败的中间状态。

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Model(&article).Updates(updates).Error; err != nil {
        return err
    }

    if err := tx.Model(&article).Association("Tags").Replace(&tags); err != nil {
        return err
    }

    return nil
})
```

事务会在下一阶段详细讲。

## 本节练习

- [ ] 创建文章时指定 `UserID` 和 `CategoryID`。
- [ ] 创建文章时同时关联多个标签。
- [ ] 给已有文章追加标签。
- [ ] 编辑文章时替换标签。
- [ ] 删除文章的某个标签关系。
- [ ] 清空文章全部标签。
- [ ] 把“更新文章 + 替换标签”放入事务。

## 常见坑

- 把删除关联误以为会删除关联对象。
- 编辑标签时用了 `Append`，导致旧标签没有被移除。
- 多步关联操作没有使用事务。
- 没有先校验标签 ID 是否真的存在。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
