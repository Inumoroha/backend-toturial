# 06 First、Take、Find 查询差异

本节目标：彻底分清 `First`、`Take`、`Last`、`Find` 的行为差异，尤其是“查不到数据”时的错误处理。

---

## 一、准备数据

```go
users := []User{
    {Username: "tom", Email: "tom@example.com", Age: 20},
    {Username: "jerry", Email: "jerry@example.com", Age: 18},
    {Username: "alice", Email: "alice@example.com", Age: 25},
}

db.Create(&users)
```

---

## 二、First

```go
var user User
err := db.First(&user).Error
```

特点：

- 默认按主键升序。
- 取第一条。
- 查不到返回 `gorm.ErrRecordNotFound`。

大致 SQL：

```sql
SELECT * FROM users ORDER BY users.id LIMIT 1;
```

根据主键查询：

```go
err := db.First(&user, 1).Error
```

---

## 三、Last

```go
var user User
err := db.Last(&user).Error
```

特点：

- 默认按主键降序。
- 取最后一条。
- 查不到返回 `gorm.ErrRecordNotFound`。

---

## 四、Take

```go
var user User
err := db.Take(&user).Error
```

特点：

- 不指定排序。
- 从数据库结果中取一条。
- 查不到返回 `gorm.ErrRecordNotFound`。

业务代码里如果关心顺序，不要用 `Take`，要写明确 `Order`。

---

## 五、Find 查询多条

```go
var users []User
err := db.Where("age >= ?", 18).Find(&users).Error
```

特点：

- 查询多条。
- 查不到通常不返回 `ErrRecordNotFound`。
- 结果是空切片。

这和 `First` 很不一样。

---

## 六、查不到数据的处理

单条查询：

```go
var user User
err := db.Where("email = ?", email).First(&user).Error
if err != nil {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, ErrUserNotFound
    }
    return nil, err
}
```

多条查询：

```go
var users []User
err := db.Where("age > ?", 100).Find(&users).Error
if err != nil {
    return nil, err
}

fmt.Println(len(users))
```

`len(users) == 0` 才表示没有结果。

---

## 七、查询条件写法

推荐：

```go
db.Where("email = ?", email).First(&user)
```

也可以：

```go
db.First(&user, "email = ?", email)
```

不要拼接：

```go
db.Where("email = '" + email + "'").First(&user)
```

---

## 八、查询目标必须匹配

查询单条用结构体：

```go
var user User
db.First(&user)
```

查询多条用切片：

```go
var users []User
db.Find(&users)
```

不要写成：

```go
var user User
db.Find(&user)
```

虽然有时能跑，但语义混乱。

---

## 九、常见错误

### 1. 用 Find 判断 ErrRecordNotFound

错误：

```go
err := db.Where("email = ?", email).Find(&users).Error
if errors.Is(err, gorm.ErrRecordNotFound) {}
```

`Find` 查不到通常不会返回这个错误。

### 2. Take 用在需要排序的场景

比如“最新文章”应该写：

```go
db.Order("created_at DESC").First(&article)
```

不要直接 `Take`。

### 3. 不处理查询不存在

登录、详情页、编辑接口都必须处理“资源不存在”。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 说清 `First`、`Last`、`Take`、`Find` 的区别。
- 正确处理 `ErrRecordNotFound`。
- 知道 `Find` 查不到时是空切片。
- 写出安全的条件查询。
- 根据业务选择合适的查询方法。
