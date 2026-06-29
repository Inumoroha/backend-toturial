# 8. DISTINCT 与 CASE WHEN：去重和条件表达式

本节目标：掌握 `DISTINCT` 的去重用法，理解 `CASE WHEN` 条件表达式，并能在查询中生成更适合后端接口返回的字段。

`DISTINCT` 和 `CASE WHEN` 都不算很大，但非常实用：

- `DISTINCT` 用来去除重复结果。
- `CASE WHEN` 用来根据条件生成不同结果。

---

## 一、DISTINCT：去重

查询所有文章作者 ID：

```sql
SELECT user_id
FROM demo.posts;
```

如果一个用户有多篇文章，`user_id` 会重复。

去重：

```sql
SELECT DISTINCT user_id
FROM demo.posts
ORDER BY user_id;
```

---

## 二、DISTINCT 多列去重

```sql
SELECT DISTINCT user_id, is_published
FROM demo.posts
ORDER BY user_id, is_published;
```

多列 `DISTINCT` 的含义是：

```text
按 user_id + is_published 的组合去重。
```

不是只对第一列去重。

---

## 三、COUNT DISTINCT

统计发布过文章的作者数量：

```sql
SELECT COUNT(DISTINCT user_id) AS author_count
FROM demo.posts;
```

统计有评论的用户数量：

```sql
SELECT COUNT(DISTINCT user_id) AS commenter_count
FROM demo.comments;
```

---

## 四、DISTINCT 和 GROUP BY 的区别

查询有哪些作者发布过文章：

```sql
SELECT DISTINCT user_id
FROM demo.posts;
```

也可以写成：

```sql
SELECT user_id
FROM demo.posts
GROUP BY user_id;
```

简单理解：

- 只想去重，用 `DISTINCT` 更直接。
- 要做分组统计，用 `GROUP BY`。

---

## 五、CASE WHEN：条件表达式

`CASE WHEN` 可以根据条件返回不同值。

查询文章状态：

```sql
SELECT
    id,
    title,
    CASE
        WHEN is_published THEN '已发布'
        ELSE '草稿'
    END AS status_text
FROM demo.posts
ORDER BY id;
```

---

## 六、多条件 CASE WHEN

根据浏览量生成热度等级：

```sql
SELECT
    p.id,
    p.title,
    s.view_count,
    CASE
        WHEN s.view_count >= 150 THEN '热门'
        WHEN s.view_count >= 80 THEN '普通'
        ELSE '冷门'
    END AS popularity
FROM demo.posts AS p
JOIN demo.post_stats AS s ON s.post_id = p.id
ORDER BY s.view_count DESC;
```

`CASE WHEN` 会从上往下判断，命中第一个条件后返回对应结果。

---

## 七、CASE WHEN 配合聚合

统计已发布和未发布文章数量：

```sql
SELECT
    COUNT(*) AS total_posts,
    SUM(CASE WHEN is_published THEN 1 ELSE 0 END) AS published_count,
    SUM(CASE WHEN NOT is_published THEN 1 ELSE 0 END) AS draft_count
FROM demo.posts;
```

这类写法常用于统计看板。

PostgreSQL 也可以使用 `FILTER` 写得更清晰：

```sql
SELECT
    COUNT(*) AS total_posts,
    COUNT(*) FILTER (WHERE is_published) AS published_count,
    COUNT(*) FILTER (WHERE NOT is_published) AS draft_count
FROM demo.posts;
```

---

## 八、常见后端场景

### 1. 给状态字段转成展示文案

```sql
SELECT
    id,
    title,
    CASE
        WHEN is_published THEN 'published'
        ELSE 'draft'
    END AS status
FROM demo.posts;
```

### 2. 给统计结果分级

```sql
SELECT
    p.id,
    p.title,
    COUNT(c.id) AS comment_count,
    CASE
        WHEN COUNT(c.id) >= 2 THEN 'active'
        ELSE 'normal'
    END AS comment_level
FROM demo.posts AS p
LEFT JOIN demo.comments AS c ON c.post_id = p.id
GROUP BY p.id, p.title
ORDER BY p.id;
```

---

## 九、常见误区

### 1. DISTINCT 是对某一列去重吗？

不一定。

如果查询多列，`DISTINCT` 是对整行结果去重。

### 2. DISTINCT 能替代 GROUP BY 吗？

只能在简单去重场景下替代。

需要统计时，还是应该使用 `GROUP BY`。

### 3. CASE WHEN 只能用于 SELECT 吗？

不是。

它也可以用于 `ORDER BY`、`WHERE` 等位置，但初学阶段先掌握在 `SELECT` 中生成字段即可。

### 4. CASE WHEN 判断顺序重要吗？

重要。

PostgreSQL 会从上到下判断，先命中的条件先返回。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 `DISTINCT` 去除重复结果。
- 理解多列 `DISTINCT` 是按整行组合去重。
- 使用 `COUNT(DISTINCT column)` 做去重统计。
- 区分 `DISTINCT` 和 `GROUP BY` 的基本用途。
- 使用 `CASE WHEN` 根据条件生成字段。
- 使用 `CASE WHEN` 做简单分级和状态转换。
- 初步理解 `CASE WHEN` 配合聚合统计的用法。