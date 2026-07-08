# 03 错误处理与 SQL 日志

本节目标：围绕「error and debug」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

GORM 的 API 大多返回 `*gorm.DB`，真正的错误在 `.Error` 字段里。学会处理错误和查看 SQL，是从“会写 demo”走向“能做项目”的关键。

## 1. 基本错误处理

```go
if err := db.Create(&user).Error; err != nil {
    return err
}
```

不要忽略错误：

```go
db.Create(&user)
```

这会让问题隐藏起来，后续排查很痛苦。

## 2. 查询不存在

```go
var user User
err := db.First(&user, "email = ?", "not-exist@example.com").Error
if err != nil {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        fmt.Println("user not found")
        return
    }
    log.Fatal(err)
}
```

需要导入：

```go
import "errors"
```

`First`、`Take`、`Last` 查询不到数据时，会返回 `gorm.ErrRecordNotFound`。

`Find` 查询不到数据时，通常不会返回 `ErrRecordNotFound`，而是返回空切片。

## 3. `First`、`Take`、`Last` 的区别

### First

按主键升序取第一条。

```go
db.First(&user)
```

### Last

按主键降序取第一条。

```go
db.Last(&user)
```

### Take

不指定排序，取一条。

```go
db.Take(&user)
```

业务代码里通常更推荐明确排序，不要依赖数据库返回顺序。

## 4. 开启 Debug

临时查看某条 SQL：

```go
db.Debug().Where("age > ?", 18).Find(&users)
```

`Debug` 适合学习和排查问题，不建议在生产所有查询上直接使用。

## 5. 配置全局日志

创建数据库连接时配置：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

需要导入：

```go
import "gorm.io/gorm/logger"
```

常见日志级别：

- `Silent`：不输出日志。
- `Error`：只输出错误。
- `Warn`：输出警告和错误。
- `Info`：输出详细 SQL。

学习阶段建议使用 `Info`，这样你能看到 GORM 生成的 SQL。

## 6. 查看影响行数

更新或删除时，经常需要判断是否真的影响了数据：

```go
result := db.Model(&User{}).
    Where("id = ?", 1).
    Update("age", 30)

if result.Error != nil {
    return result.Error
}

if result.RowsAffected == 0 {
    return fmt.Errorf("user not found")
}
```

`RowsAffected` 是真实项目里很常用的判断条件。

## 7. 本节练习

- [ ] 查询一个不存在的用户，正确处理 `ErrRecordNotFound`。
- [ ] 对比 `First` 和 `Find` 查询不到数据时的行为。
- [ ] 用 `Debug` 打印一条查询 SQL。
- [ ] 配置全局 `logger.Info`。
- [ ] 更新一个不存在的用户，观察 `RowsAffected`。

## 小结

初学 GORM 时，每条数据库操作都应该做到三件事：

```text
检查 Error
观察 SQL
关注 RowsAffected
```
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
