# 2. jsonb 类型基础与用户偏好设置

本节目标：理解 `jsonb` 的作用，掌握 JSONB 的插入、查询、更新和基本约束，并能用它存储用户偏好设置这类扩展配置。

`jsonb` 是 PostgreSQL 中非常强大的类型。

它可以存储 JSON 数据，例如：

```json
{
  "theme": "dark",
  "language": "zh-CN",
  "notifications": {
    "email": true,
    "sms": false
  },
  "shortcuts": ["search", "new_article"]
}
```

和普通 `text` 不同，`jsonb` 不是把 JSON 当纯字符串存起来。PostgreSQL 会以二进制形式存储它，并支持查询内部字段、建立索引和做包含判断。

---

## 一、json 和 jsonb 的区别

PostgreSQL 有两个 JSON 类型：

- `json`
- `jsonb`

初学阶段可以先记住：

```text
大多数业务场景优先使用 jsonb。
```

原因是：

- `jsonb` 支持更多索引能力。
- `jsonb` 查询和比较更方便。
- `jsonb` 会解析并规范化 JSON 内容。

`json` 会保留原始文本格式，例如空格、字段顺序等。大多数后端业务并不需要保留这些细节。

---

## 二、适合 jsonb 的场景

适合：

- 用户偏好设置。
- 页面布局配置。
- 第三方 API 回调原始内容。
- 扩展字段。
- 低频查询的灵活属性。
- 结构可能变化，但仍然希望放在数据库里的配置。

例如用户偏好设置：

```text
主题：dark/light
语言：zh-CN/en-US
通知方式：email/sms
快捷入口：数组
```

这些字段可能不断增加，且通常不参与复杂关联查询。用 JSONB 很合适。

---

## 三、创建用户偏好设置表

```sql
CREATE TABLE demo.jsonb_user_preferences (
    user_id bigint PRIMARY KEY,
    preferences jsonb NOT NULL DEFAULT '{}'::jsonb,
    updated_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT jsonb_user_preferences_is_object_check
        CHECK (jsonb_typeof(preferences) = 'object')
);
```

这里的约束：

```sql
CHECK (jsonb_typeof(preferences) = 'object')
```

表示 `preferences` 必须是 JSON 对象，不能是数组、字符串或数字。

有效：

```json
{"theme": "dark"}
```

无效：

```json
["theme", "dark"]
```

---

## 四、插入 JSONB 数据

```sql
INSERT INTO demo.jsonb_user_preferences (user_id, preferences)
VALUES
    (
        1,
        '{
            "theme": "dark",
            "language": "zh-CN",
            "notifications": {
                "email": true,
                "sms": false
            },
            "shortcuts": ["search", "new_article"]
        }'
    );
```

注意字符串要是合法 JSON：

- 字段名必须使用双引号。
- 字符串值必须使用双引号。
- 布尔值是 `true` / `false`。
- 不能有多余逗号。

---

## 五、查询 JSONB 字段

### 1. 使用 `->` 取 JSON 值

```sql
SELECT preferences -> 'theme' AS theme
FROM demo.jsonb_user_preferences
WHERE user_id = 1;
```

返回仍然是 JSONB：

```json
"dark"
```

### 2. 使用 `->>` 取文本值

```sql
SELECT preferences ->> 'theme' AS theme
FROM demo.jsonb_user_preferences
WHERE user_id = 1;
```

返回文本：

```text
dark
```

简单记法：

```text
->  返回 JSON/JSONB
->> 返回 text
```

### 3. 查询嵌套字段

```sql
SELECT preferences -> 'notifications' ->> 'email' AS email_enabled
FROM demo.jsonb_user_preferences
WHERE user_id = 1;
```

也可以写成：

```sql
SELECT preferences #>> '{notifications,email}' AS email_enabled
FROM demo.jsonb_user_preferences
WHERE user_id = 1;
```

`#>>` 适合按路径取文本。

---

## 六、根据 JSONB 内容过滤

查询主题为 dark 的用户：

```sql
SELECT user_id, preferences
FROM demo.jsonb_user_preferences
WHERE preferences ->> 'theme' = 'dark';
```

查询开启邮件通知的用户：

```sql
SELECT user_id, preferences
FROM demo.jsonb_user_preferences
WHERE preferences #>> '{notifications,email}' = 'true';
```

注意这里取出来的是文本 `'true'`。

如果想按 JSONB 包含查询，可以写：

```sql
SELECT user_id, preferences
FROM demo.jsonb_user_preferences
WHERE preferences @> '{"theme": "dark"}'::jsonb;
```

`@>` 表示左边 JSONB 是否包含右边 JSONB。

---

## 七、判断 key 是否存在

```sql
SELECT user_id
FROM demo.jsonb_user_preferences
WHERE preferences ? 'theme';
```

`?` 表示顶层 key 是否存在。

判断是否包含任意一个 key：

```sql
SELECT user_id
FROM demo.jsonb_user_preferences
WHERE preferences ?| ARRAY['theme', 'language'];
```

判断是否包含所有 key：

```sql
SELECT user_id
FROM demo.jsonb_user_preferences
WHERE preferences ?& ARRAY['theme', 'language'];
```

---

## 八、更新 JSONB 字段

### 1. 替换整个 JSONB

```sql
UPDATE demo.jsonb_user_preferences
SET
    preferences = '{
        "theme": "light",
        "language": "zh-CN"
    }'::jsonb,
    updated_at = now()
WHERE user_id = 1;
```

这种方式简单，但会覆盖原有所有字段。

### 2. 更新某个路径

使用 `jsonb_set`：

```sql
UPDATE demo.jsonb_user_preferences
SET
    preferences = jsonb_set(
        preferences,
        '{theme}',
        '"light"'::jsonb
    ),
    updated_at = now()
WHERE user_id = 1;
```

路径：

```text
{theme}
```

表示更新顶层 `theme`。

更新嵌套字段：

```sql
UPDATE demo.jsonb_user_preferences
SET
    preferences = jsonb_set(
        preferences,
        '{notifications,email}',
        'false'::jsonb
    ),
    updated_at = now()
WHERE user_id = 1;
```

### 3. 增加或合并字段

```sql
UPDATE demo.jsonb_user_preferences
SET
    preferences = preferences || '{"timezone": "Asia/Shanghai"}'::jsonb,
    updated_at = now()
WHERE user_id = 1;
```

`||` 可以合并 JSONB 对象。相同 key 时，右边覆盖左边。

---

## 九、删除 JSONB 字段

删除顶层 key：

```sql
UPDATE demo.jsonb_user_preferences
SET
    preferences = preferences - 'timezone',
    updated_at = now()
WHERE user_id = 1;
```

删除嵌套字段可以使用路径删除：

```sql
UPDATE demo.jsonb_user_preferences
SET
    preferences = preferences #- '{notifications,sms}',
    updated_at = now()
WHERE user_id = 1;
```

---

## 十、JSONB 字段约束

JSONB 灵活，但灵活也意味着容易失控。

可以加一些基础约束。

### 1. 限制必须是对象

```sql
CHECK (jsonb_typeof(preferences) = 'object')
```

### 2. 限制某个字段的值

```sql
ALTER TABLE demo.jsonb_user_preferences
ADD CONSTRAINT jsonb_user_preferences_theme_check
CHECK (
    preferences ->> 'theme' IS NULL
    OR preferences ->> 'theme' IN ('light', 'dark')
);
```

这个约束表示：

```text
theme 可以不存在。
如果存在，只能是 light 或 dark。
```

### 3. 限制某个字段类型

```sql
ALTER TABLE demo.jsonb_user_preferences
ADD CONSTRAINT jsonb_user_preferences_notifications_check
CHECK (
    preferences -> 'notifications' IS NULL
    OR jsonb_typeof(preferences -> 'notifications') = 'object'
);
```

---

## 十一、JSONB 的更新成本

虽然可以局部更新 JSONB 中的某个字段，但从存储角度看，更新 JSONB 仍然可能导致整行产生新版本。

所以不要把一个巨大 JSON 文档频繁更新。

如果某些字段：

- 很大。
- 高频更新。
- 高频查询。
- 需要强约束。

就应该考虑拆成普通列或单独表。

---

## 十二、常见误区

### 1. JSONB 可以替代所有表结构吗？

不能。

JSONB 适合半结构化和扩展配置，不适合逃避关系建模。

### 2. JSONB 里什么都能放，所以不用约束？

不是。

关键配置仍然应该加必要约束，避免脏数据。

### 3. `->` 和 `->>` 一样吗？

不一样。

`->` 返回 JSONB，`->>` 返回文本。

### 4. JSONB 更新一定很便宜吗？

不一定。

频繁更新大 JSONB 字段可能带来存储和性能成本。

---

## 十三、本节达标标准

学完本节后，你应该能够做到：

- 创建 `jsonb` 字段。
- 使用 JSONB 存储用户偏好设置。
- 使用 `->`、`->>`、`#>>` 查询 JSONB 内容。
- 使用 `@>` 做包含查询。
- 使用 `?` 判断 key 是否存在。
- 使用 `jsonb_set` 更新 JSONB 路径。
- 使用 `||` 合并 JSONB 对象。
- 为 JSONB 字段添加基础 `CHECK` 约束。
- 知道 JSONB 灵活但不能替代表设计。
