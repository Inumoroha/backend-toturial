# 05 Create 与批量插入

本节目标：系统掌握 GORM 的新增数据能力，包括单条插入、批量插入、默认值、主键回填、选择字段插入和常见唯一索引冲突。

---

## 一、准备模型

```go
type User struct {
    ID        uint      `gorm:"primaryKey"`
    Username  string    `gorm:"size:64;not null;uniqueIndex"`
    Email     string    `gorm:"size:128;not null;uniqueIndex"`
    Age       int       `gorm:"not null;default:18"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

迁移：

```go
db.AutoMigrate(&User{})
```

---

## 二、单条插入

```go
user := User{
    Username: "tom",
    Email:    "tom@example.com",
    Age:      20,
}

err := db.Create(&user).Error
if err != nil {
    return err
}

fmt.Println(user.ID)
```

执行成功后，`user.ID` 会被回填。

大致 SQL：

```sql
INSERT INTO users (username, email, age, created_at, updated_at)
VALUES (...);
```

---

## 三、主键回填为什么重要

很多业务需要插入后立刻使用主键：

```text
先创建订单
再创建订单明细
订单明细需要 order.ID
```

如果不知道 GORM 会回填主键，后续写事务时会很别扭。

---

## 四、批量插入

```go
users := []User{
    {Username: "u1", Email: "u1@example.com", Age: 18},
    {Username: "u2", Email: "u2@example.com", Age: 20},
    {Username: "u3", Email: "u3@example.com", Age: 22},
}

if err := db.Create(&users).Error; err != nil {
    return err
}
```

插入后，切片里的每个元素也会尽量回填主键：

```go
for _, user := range users {
    fmt.Println(user.ID)
}
```

---

## 五、指定批大小

大量插入时：

```go
err := db.CreateInBatches(users, 100).Error
```

含义：每 100 条作为一批插入。

批大小不是越大越好。字段很多、网络较慢、数据库参数限制较严时，过大的批次可能失败。

---

## 六、选择字段插入

只插入部分字段：

```go
user := User{
    Username: "select_user",
    Email:    "select@example.com",
    Age:      30,
}

err := db.Select("Username", "Email").Create(&user).Error
```

这里 `Age` 不会显式插入，数据库或 GORM 默认值会起作用。

---

## 七、忽略字段插入

```go
err := db.Omit("Age").Create(&user).Error
```

适合某些字段由数据库默认值决定的场景。

---

## 八、唯一索引冲突

如果邮箱唯一：

```go
Email string `gorm:"uniqueIndex"`
```

重复插入会报错。

练习时可以故意执行两次：

```go
db.Create(&User{Username: "tom1", Email: "same@example.com"})
db.Create(&User{Username: "tom2", Email: "same@example.com"})
```

第二次应该失败。

真实项目里，注册用户时要处理这个错误，返回“邮箱已存在”。

---

## 九、常见错误

### 1. 传值而不是指针

错误：

```go
db.Create(user)
```

正确：

```go
db.Create(&user)
```

如果不传指针，主键无法正常回填。

### 2. 忽略错误

错误：

```go
db.Create(&user)
```

正确：

```go
if err := db.Create(&user).Error; err != nil {
    return err
}
```

### 3. 批量插入数据重复

如果批量数据里有重复唯一字段，整批可能失败。导入数据前要清洗。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 `Create` 插入单条记录。
- 解释主键回填。
- 使用 `Create` 批量插入切片。
- 使用 `CreateInBatches` 指定批大小。
- 使用 `Select` 和 `Omit` 控制插入字段。
- 处理唯一索引冲突。
