# 5. EXISTS 与 IN：判断关联数据是否存在

本节目标：理解 `IN`、`EXISTS`、`NOT IN`、`NOT EXISTS` 的用法，能够判断某条数据是否存在关联记录，并知道查询“没有关联数据”时为什么更推荐 `NOT EXISTS`。

`IN` 和 `EXISTS` 都常用于“根据另一个查询结果过滤数据”。它们在后端业务中很常见，例如：

- 查询有评论的文章。
- 查询没有评论的文章。
- 查询发布过文章的用户。
- 查询没有下过订单的用户。

---

## 一、IN 是什么

`IN` 用来判断某个值是否在一组结果中。

查询 `zhangsan` 和 `lisi` 发布的文章：

```sql
SELECT id, title, user_id
FROM demo.posts
WHERE user_id IN (
    SELECT id
    FROM demo.users
    WHERE username IN ('zhangsan', 'lisi')
)
ORDER BY id;
```

内层查询返回多个用户 ID，外层查询筛选 `user_id` 在这些 ID 中的文章。

---

## 二、EXISTS 是什么

`EXISTS` 用来判断子查询是否至少返回一行。

查询有评论的文章：

```sql
SELECT p.id, p.title
FROM demo.posts AS p
WHERE EXISTS (
    SELECT 1
    FROM demo.comments AS c
    WHERE c.post_id = p.id
)
ORDER BY p.id;
```

`EXISTS` 不关心子查询返回什么列，只关心有没有结果，所以常写 `SELECT 1`。

---

## 三、NOT EXISTS：查询没有关联数据

查询没有评论的文章：

```sql
SELECT p.id, p.title
FROM demo.posts AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM demo.comments AS c
    WHERE c.post_id = p.id
)
ORDER BY p.id;
```

这类查询非常常见：

```text
没有评论的文章
没有文章的用户
没有标签的文章
没有订单的用户
```

---

## 四、NOT IN 的 NULL 问题

`NOT IN` 遇到 `NULL` 时容易出现不符合直觉的结果。

例如：

```sql
SELECT id, title
FROM demo.posts
WHERE id NOT IN (
    SELECT post_id
    FROM demo.comments
);
```

如果子查询结果里包含 `NULL`，整个判断可能无法得到你想要的结果。

所以查询“没有关联数据”时，更推荐：

```sql
NOT EXISTS
```

或者：

```sql
LEFT JOIN ... WHERE right_table.id IS NULL
```

---

## 五、IN、EXISTS、JOIN 怎么选

简单建议：

| 场景 | 推荐写法 |
| --- | --- |
| 判断某列是否在一组值里 | `IN` |
| 判断关联数据是否存在 | `EXISTS` |
| 判断关联数据不存在 | `NOT EXISTS` |
| 需要同时返回多张表字段 | `JOIN` |

查询发布过文章的用户：

```sql
SELECT u.id, u.username
FROM demo.users AS u
WHERE EXISTS (
    SELECT 1
    FROM demo.posts AS p
    WHERE p.user_id = u.id
)
ORDER BY u.id;
```

查询没有发布文章的用户：

```sql
SELECT u.id, u.username
FROM demo.users AS u
WHERE NOT EXISTS (
    SELECT 1
    FROM demo.posts AS p
    WHERE p.user_id = u.id
)
ORDER BY u.id;
```

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 使用 `IN` 根据子查询结果过滤数据。
- 使用 `EXISTS` 判断关联数据是否存在。
- 使用 `NOT EXISTS` 查询没有关联数据的记录。
- 知道 `NOT IN` 遇到 `NULL` 时可能出问题。
- 能在 `IN`、`EXISTS`、`JOIN` 之间做基础选择。