# 09 单表 CRUD 综合练习

本节目标：用一个 `users` 表完成完整 CRUD 练习，并把新增、查询、更新、删除、错误处理、SQL 日志全部串起来。

---

## 一、练习目标

你要实现一个命令行程序，按顺序执行：

```text
1. 自动迁移 users 表
2. 清理旧测试数据
3. 批量创建用户
4. 根据邮箱查询用户
5. 分页查询用户
6. 更新用户年龄
7. 把用户年龄更新为 0
8. 删除用户
9. 查询不存在用户并处理错误
```

---

## 二、文件建议

```text
code/gorm-study
├── go.mod
└── main.go
```

初学阶段先用一个文件写完整流程，确认理解后再拆分。

---

## 三、模型

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Username  string         `gorm:"size:64;not null;uniqueIndex"`
    Email     string         `gorm:"size:128;not null;uniqueIndex"`
    Age       int            `gorm:"not null;default:18"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

---

## 四、核心练习代码

```go
func run(db *gorm.DB) error {
    if err := db.AutoMigrate(&User{}); err != nil {
        return err
    }

    if err := db.Unscoped().
        Where("email LIKE ?", "%@crud.local").
        Delete(&User{}).Error; err != nil {
        return err
    }

    users := []User{
        {Username: "tom", Email: "tom@crud.local", Age: 20},
        {Username: "jerry", Email: "jerry@crud.local", Age: 18},
        {Username: "alice", Email: "alice@crud.local", Age: 25},
    }
    if err := db.Create(&users).Error; err != nil {
        return err
    }

    var tom User
    if err := db.Where("email = ?", "tom@crud.local").First(&tom).Error; err != nil {
        return err
    }

    var pageUsers []User
    if err := db.Order("id DESC").Limit(2).Offset(0).Find(&pageUsers).Error; err != nil {
        return err
    }

    result := db.Model(&User{}).Where("id = ?", tom.ID).Update("age", 21)
    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return fmt.Errorf("user not found")
    }

    if err := db.Model(&tom).Select("age").Updates(User{Age: 0}).Error; err != nil {
        return err
    }

    if err := db.Delete(&tom).Error; err != nil {
        return err
    }

    var missing User
    err := db.Where("email = ?", "missing@crud.local").First(&missing).Error
    if err != nil && errors.Is(err, gorm.ErrRecordNotFound) {
        fmt.Println("missing user not found")
        return nil
    }

    return err
}
```

---

## 五、观察 SQL

打开日志：

```go
Logger: logger.Default.LogMode(logger.Info)
```

运行后观察：

- `AutoMigrate` 是否检查或创建表。
- `Create` 是否批量插入。
- `First` 是否带 `ORDER BY id LIMIT 1`。
- `Find` 是否带 `LIMIT`。
- 软删除是否是 `UPDATE deleted_at`。
- `Unscoped().Delete` 是否是真正 `DELETE`。

---

## 六、数据库验证

查询用户：

```sql
SELECT id, username, email, age, deleted_at
FROM users
WHERE email LIKE '%@crud.local'
ORDER BY id DESC;
```

因为普通删除是软删除，你应该能看到 `deleted_at` 不为空的数据。

如果清理用了：

```go
Unscoped().Delete
```

则是物理删除。

---

## 七、常见错误

### 1. 忘记 Unscoped 清理测试数据

软删除数据仍然占用唯一索引，可能导致重复邮箱插入失败。

### 2. 结构体更新 0 失败

要用：

```go
Select("age")
```

或者用 `map`。

### 3. 不看 SQL 日志

这节练习的重点不是跑通，而是理解每一步 SQL。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 独立写出单表 CRUD 程序。
- 使用批量创建。
- 使用分页查询。
- 正确更新零值。
- 区分软删除和物理删除。
- 处理查询不存在。
- 通过数据库客户端验证结果。
