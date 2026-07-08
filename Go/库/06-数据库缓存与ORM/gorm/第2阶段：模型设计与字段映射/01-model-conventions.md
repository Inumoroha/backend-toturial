# 01 模型约定与字段映射

本节目标：围绕「model conventions」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

GORM 的模型通常是 Go 结构体。结构体字段映射到数据库列，结构体实例映射到数据库记录。

## 1. 默认命名约定

```go
type User struct {
    ID    uint
    Name  string
    Email string
}
```

默认情况下：

- `User` 映射到表 `users`。
- `ID` 映射到列 `id`，并作为主键。
- `Name` 映射到列 `name`。
- `Email` 映射到列 `email`。

GORM 会把结构体名转为复数蛇形表名，把字段名转为蛇形列名。

例如：

```go
type BlogArticle struct {
    CreatedAt time.Time
}
```

默认映射：

```text
blog_articles.created_at
```

## 2. `gorm.Model`

GORM 提供了一个常用基础模型：

```go
type Model struct {
    ID        uint `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

使用方式：

```go
type User struct {
    gorm.Model
    Name  string
    Email string
}
```

等价于你的模型拥有：

- `ID`
- `CreatedAt`
- `UpdatedAt`
- `DeletedAt`

初学时可以用 `gorm.Model`，但真实项目里我更建议显式写字段，这样字段类型、JSON 输出、数据库含义更清楚。

## 3. 显式模型写法

```go
type User struct {
    ID        uint           `gorm:"primaryKey"`
    Name      string         `gorm:"size:64;not null"`
    Email     string         `gorm:"size:128;not null;uniqueIndex"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

这种写法更适合学习，因为每个字段都看得见。

## 4. 自定义表名

如果你不想使用默认表名，可以实现 `TableName` 方法：

```go
func (User) TableName() string {
    return "app_users"
}
```

之后 GORM 会把 `User` 映射到 `app_users` 表。

注意：表名方法通常不要写复杂逻辑。大多数项目里表名应该稳定。

## 5. 忽略字段

如果某个字段只在 Go 代码里使用，不想映射到数据库：

```go
type User struct {
    ID       uint
    Name     string
    Password string
    Token    string `gorm:"-"`
}
```

`Token` 不会被建表，也不会被读写数据库。

## 6. 嵌入结构体

你可以把通用字段抽出来：

```go
type Timestamp struct {
    CreatedAt time.Time
    UpdatedAt time.Time
}

type User struct {
    ID uint
    Timestamp
    Name string
}
```

GORM 会把嵌入结构体的字段展开到当前表。

## 7. 时间字段

如果模型里有这些字段，GORM 会自动维护：

```go
CreatedAt time.Time
UpdatedAt time.Time
```

新增时：

- 自动设置 `CreatedAt`
- 自动设置 `UpdatedAt`

更新时：

- 自动更新 `UpdatedAt`

## 本节练习

- [ ] 定义 `User` 模型。
- [ ] 用默认表名创建表。
- [ ] 改用 `TableName` 自定义表名。
- [ ] 添加 `CreatedAt` 和 `UpdatedAt`。
- [ ] 添加一个 `gorm:"-"` 字段，观察数据库表是否创建该列。

## 检查点

你应该能回答：

- `UserProfile` 默认表名是什么？
- `CreatedAt` 默认列名是什么？
- `gorm.Model` 包含哪些字段？
- 什么场景适合自定义表名？
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
