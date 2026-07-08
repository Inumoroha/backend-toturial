# 2. message 字段类型与默认值

本节目标：掌握常用字段类型、repeated、map、enum，以及 proto3 默认值带来的影响。

Protobuf message 是接口数据结构。字段类型选择会影响生成代码、兼容性、表达能力和业务语义。

proto3 有一个重要特点：很多标量字段没有“是否设置”的信息，只能看到默认值。这一点在设计更新接口时尤其重要。

---

## 一、核心直觉

- 常用整数：`int32`、`int64`、`uint64`，业务 ID 常用 `int64` 或 `string`。
- 常用文本：`string`。
- 布尔值：`bool` 默认是 `false`。
- 列表：`repeated`。
- 字典：`map<string, string>` 这类形式。
- 枚举：`enum` 的第一个值通常是 `UNKNOWN`，编号为 0。
- 需要表达“未设置”时，可以考虑 `optional` 或 wrapper 类型。

---

## 二、动手步骤

1. 给 `User` 增加状态枚举。
2. 给列表响应增加 `repeated User users`。
3. 给用户扩展属性增加 `map<string, string> labels`。
4. 思考更新用户时，空字符串代表清空，还是代表未传？

---

## 三、参考代码或命令

```proto
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_DISABLED = 2;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  UserStatus status = 4;
  repeated string roles = 5;
  map<string, string> labels = 6;
}
```

---

## 四、常见问题

- enum 第一个值不是 0：proto3 要求第一个枚举值编号为 0。
- 用 `bool` 表示三态：`false` 无法区分“没传”和“传了 false”。
- 过度使用 map：map 灵活但弱约束，核心字段还是建议明确建模。

---

## 五、练习任务

- 设计一个 `Product` message，包含 id、name、price、tags、status。
- 思考 `price` 用 `int64` 表示分，还是用 `double` 表示元，写下理由。
- 设计一个局部更新请求，说明哪些字段可能需要 optional。

---

## 六、完成标准

- 能为常见业务字段选择合适类型。
- 知道 proto3 默认值的影响。
- 能正确使用 repeated、map、enum。


---

## 七、常用类型选择建议

### ID 类型

常见选择：

```proto
int64 id = 1;
string id = 1;
```

如果你的系统 ID 来自数据库自增或雪花算法，可以用 `int64`。如果 ID 是 UUID、ULID、外部平台 ID，可以用 `string`。

不要在同一套 proto 中有的服务用 int64，有的服务用 string，除非有明确原因。

### 金额类型

不要优先使用 `double` 表示金额。推荐用最小货币单位：

```proto
int64 amount_cents = 1;
```

例如 12.34 元表示为 1234 分。这样可以避免浮点精度问题。

### 时间类型

学习阶段可以先用 string：

```proto
string created_at = 1;
```

真实项目更推荐使用标准类型：

```proto
import "google/protobuf/timestamp.proto";

google.protobuf.Timestamp created_at = 1;
```

---

## 八、proto3 默认值的影响

proto3 中标量类型有默认值：

| 类型 | 默认值 |
| --- | --- |
| string | 空字符串 |
| int32/int64 | 0 |
| bool | false |
| enum | 第一个枚举值，通常是 0 |
| repeated | 空列表 |
| map | 空 map |

这会带来一个问题：你有时无法区分“客户端没传”和“客户端传了默认值”。

例如：

```proto
message UpdateUserRequest {
  int64 id = 1;
  string name = 2;
  bool active = 3;
}
```

如果 `active` 是 false，你不知道客户端是想设置为 false，还是根本没传。局部更新场景要特别小心。

---

## 九、optional 的使用场景

如果确实需要知道字段是否设置，可以考虑：

```proto
message UpdateUserRequest {
  int64 id = 1;
  optional string name = 2;
  optional bool active = 3;
}
```

生成 Go 代码后，optional 字段通常是指针或带 presence 信息的形式，可以判断是否设置。

不要滥用 optional。普通创建、查询请求大多不需要。

---

## 十、repeated 和 map

### repeated

```proto
message ListUsersResponse {
  repeated User users = 1;
}
```

适合有序或可重复的列表。

### map

```proto
message User {
  map<string, string> labels = 1;
}
```

适合扩展属性、标签、元数据。

但是核心业务字段不要随便放进 map。比如用户姓名、邮箱、状态应该明确建字段，而不是：

```proto
map<string, string> fields = 1;
```

这种设计会丢失类型约束。

---

## 十一、enum 设计规范

推荐写法：

```proto
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_DISABLED = 2;
}
```

原因：

- proto3 要求第一个枚举值是 0。
- 0 值代表未指定，比直接 ACTIVE 更安全。
- 枚举值带前缀，避免不同 enum 之间命名冲突。

不要写：

```proto
enum Status {
  ACTIVE = 0;
  DISABLED = 1;
}
```

这个设计没有给“未知/未指定”留空间。

---

## 十二、练习：设计 Product message

请写一个商品结构：

```proto
message Product {
  int64 id = 1;
  string name = 2;
  int64 price_cents = 3;
  int32 stock = 4;
  ProductStatus status = 5;
  repeated string tags = 6;
}
```

再设计枚举：

```proto
enum ProductStatus {
  PRODUCT_STATUS_UNSPECIFIED = 0;
  PRODUCT_STATUS_ON_SHELF = 1;
  PRODUCT_STATUS_OFF_SHELF = 2;
}
```

思考：

- 为什么价格用 int64？
- 为什么 status 第一个值不是 ON_SHELF？
- tags 为什么用 repeated？

---

## 十三、完成标准

你应该能为常见业务字段做出合理选择：

```text
ID -> int64 或 string
金额 -> int64 cents
状态 -> enum
列表 -> repeated
扩展标签 -> map
局部更新 -> optional 或专门设计 FieldMask
```
---

## 教程闭环检查

为了保证本节不是只停留在概念层面，学习时请按下面闭环完成：

1. **完整操作步骤**：先按正文顺序完成本节涉及的环境检查、文件创建、proto 编写、代码生成或运行验证。
2. **完整代码或命令**：本节如果涉及代码，请使用正文中的完整示例；如果是概念准备章节，请至少执行或记录正文给出的检查命令。
3. **运行命令**：把本节出现的关键命令实际运行一遍，例如 `go version`、`protoc --version`、`protoc ...`、`go run ...` 或 `grpcurl ...`。
4. **预期输出**：运行后对照正文中的预期输出；如果输出不同，先不要跳到下一节。
5. **常见错误排查**：遇到 PATH、go_package、端口占用、连接失败、生成文件缺失等问题时，优先按本节排错思路定位。
6. **练习任务**：完成正文中的练习，不只复制代码，要至少做一次参数或字段修改。
7. **完成标准**：能不看教程复述本节做了什么、为什么这样做、出错时从哪里查，才算真正完成。