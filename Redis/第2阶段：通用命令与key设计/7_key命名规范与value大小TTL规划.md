# 7. key 命名规范与 value 大小、TTL 规划

前面几节我们学习了基础命令、过期时间、计数器、批量读写、安全遍历和危险命令。

这一节把它们收束到真正的工程问题：如何设计清晰、可维护、不容易冲突的 Redis key，以及如何规划 value 大小和 TTL。

这是第二阶段最重要的一节。

学完这一节后，你应该能够：

- 设计有业务含义的 Redis key。
- 避免 key 命名冲突和风格混乱。
- 根据场景选择合理前缀和层级。
- 控制 value 大小，避免 Big Key。
- 为不同类型的数据规划 TTL。
- 写出一份项目级 Redis key 规范。

---

## 一、为什么 key 设计很重要

Redis 不像关系型数据库那样有表结构、字段约束和外键关系。

很多时候，你面对的是一堆 key：

```text
user:profile:1001
article:detail:9527
login:token:user:1001
rate_limit:login:ip:127.0.0.1
```

如果 key 设计清晰，排查问题会很轻松。

如果 key 设计混乱，Redis 会变成一团很难维护的临时字符串仓库。

糟糕的 key 设计会带来：

- 不知道 key 属于哪个业务。
- 不知道 value 里保存什么。
- 多个模块使用同一个 key 导致覆盖。
- 无法按前缀安全扫描。
- TTL 规划混乱。
- 缓存删除时找不到准确 key。
- 线上排查成本高。

所以 Redis 不是随手 `SET a 1` 就完事。真正重要的是 key 设计。

---

## 二、推荐命名风格

推荐使用冒号分隔层级：

```text
业务:对象:标识:字段或用途
```

常见例子：

```text
user:profile:1001
article:detail:9527
article:view_count:9527
login:token:user:1001
sms:code:phone:13800138000
rate_limit:login:ip:127.0.0.1
```

这种命名方式有几个好处：

- 可读性强。
- 方便按前缀扫描。
- 图形化客户端通常会按冒号展示层级。
- 不同业务模块不容易冲突。

注意：冒号只是命名约定，不是 Redis 真实目录。

---

## 三、命名要包含足够上下文

不推荐：

```text
token:1001
code:13800138000
count:1001
cache:9527
```

这些 key 太模糊。

更推荐：

```text
login:token:user:1001
sms:code:phone:13800138000
article:view_count:1001
article:detail:9527
```

一个好 key 应该能回答几个问题：

- 属于哪个业务？
- 保存什么对象？
- 对象 ID 是什么？
- 这个 key 的用途是什么？

比如：

```text
article:detail:9527
```

一眼能看出：这是文章详情，文章 ID 是 9527。

---

## 四、保持同一业务下风格一致

不要在同一个项目里同时出现：

```text
user:profile:1001
user_profile_1002
profile:user:1003
user:1004:profile
```

这些 key 都可能表示用户资料，但风格不一致。

这会让扫描、删除、排查都变麻烦。

应该统一为一种风格，例如：

```text
user:profile:{user_id}
```

具体实例：

```text
user:profile:1001
user:profile:1002
user:profile:1003
```

后端代码里也应该集中封装 key 生成函数。

伪代码：

```text
UserProfileKey(userID) = "user:profile:" + userID
ArticleDetailKey(articleID) = "article:detail:" + articleID
```

不要在各个业务函数里到处手写 key 字符串。

---

## 五、key 不要太短，也不要太长

太短的问题是语义不清：

```text
u:1
p:9527
c:abc
```

排查时很难看懂。

太长的问题是浪费内存，也让日志和排查不方便：

```text
production:backend:article:service:cache:article:detail:article_id:9527:version:v1
```

更平衡的写法：

```text
article:detail:9527
```

如果确实需要区分环境或系统，可以在部署层隔离 Redis，或加简短前缀：

```text
prod:article:detail:9527
```

但不要把所有信息都塞进 key。

---

## 六、是否要把环境名写进 key

这取决于部署方式。

更推荐：不同环境使用不同 Redis 实例或不同数据库。

比如：

```text
开发环境 Redis
测试环境 Redis
生产环境 Redis
```

这样 key 可以保持简洁：

```text
article:detail:9527
```

如果多个环境不得不共用同一个 Redis，才考虑加环境前缀：

```text
dev:article:detail:9527
test:article:detail:9527
```

但共用 Redis 容易误操作，不是理想方案。

---

## 七、是否要把版本号写进 key

有时缓存结构会变化。

比如旧 value：

```json
{"id":1001,"title":"Redis"}
```

新 value：

```json
{"id":1001,"title":"Redis","author":{"id":1,"name":"tom"}}
```

如果新旧结构不兼容，可以给 key 加版本：

```text
article:detail:v2:1001
```

这样新代码读新 key，旧缓存自然过期。

但不要滥用版本号。只有 value 结构发生不兼容变化，或者需要灰度切换时，版本号才有意义。

---

## 八、常见业务 key 设计示例

用户资料：

```text
user:profile:{user_id}
```

文章详情：

```text
article:detail:{article_id}
```

文章阅读量：

```text
article:view_count:{article_id}
```

用户登录 token：

```text
login:token:user:{user_id}
```

短信验证码：

```text
sms:code:phone:{phone}
```

接口限流：

```text
rate_limit:{api}:{dimension}:{value}
```

例如：

```text
rate_limit:login:ip:127.0.0.1
rate_limit:sms:phone:13800138000
```

短链接详情：

```text
shortlink:detail:{code}
```

空值缓存：

```text
shortlink:not_found:{code}
```

排行榜：

```text
rank:article:daily:{yyyyMMdd}
```

---

## 九、value 大小为什么要控制

Redis 数据主要在内存里，value 太大会带来很多问题。

Big Key 可能导致：

- 占用大量内存。
- 网络传输变慢。
- `GET` 返回大 value 时拖慢请求。
- `DEL` 删除大 key 时阻塞。
- 持久化和复制压力变大。
- 客户端反序列化耗时增加。

比如一个 key 保存 20 MB JSON：

```text
article:all:details -> 超大 JSON 数组
```

看起来减少了 key 数量，但每次读取都要传输巨大内容。只想查一篇文章，也要拿回所有文章。

更合理的设计：

```text
article:detail:1001
article:detail:1002
article:detail:1003
```

按对象拆分，读取更精确，也方便单独删除和设置 TTL。

---

## 十、如何控制 value 大小

几个实用原则：

- 不要把整个表的数据塞进一个 key。
- 不要把无限增长的列表放在一个 key 里。
- 缓存对象时只缓存接口需要的字段。
- 大文本、大图片、大文件不要直接放 Redis。
- 列表、集合、排行榜要控制长度。
- 对可能增长的数据结构设置清理策略。

例如文章列表缓存，不建议：

```text
article:list:all -> 包含所有文章的大 JSON
```

更合理：

```text
article:list:page:1:size:20
article:list:page:2:size:20
article:detail:1001
```

列表页缓存列表数据，详情页缓存详情数据，各自设置 TTL。

---

## 十一、TTL 规划的基本原则

TTL 要结合业务变化频率和一致性要求。

你可以按这几个问题思考：

1. 这个数据是不是临时数据？
2. 这个数据能不能长期不更新？
3. 用户能不能接受短时间旧数据？
4. 数据库压力是否很大？
5. key 数量是否会持续增长？
6. 这个 key 如果不删除，会不会有安全风险？

不同场景 TTL 不同：

| 场景 | 建议 TTL 思路 |
| --- | --- |
| 短信验证码 | 短，通常几分钟 |
| 登录 token | 根据登录策略，几小时到几天 |
| 文章详情 | 中等，几分钟到几十分钟 |
| 商品库存 | 很短，或不简单缓存 |
| 空值缓存 | 很短，防穿透但避免长时间误伤 |
| 限流计数 | 等于限流窗口 |
| 排行榜 | 按统计周期，日榜可按天 |

---

## 十二、缓存 TTL 不要全部一样

如果大量缓存同时写入，并设置完全相同 TTL，它们可能同时过期。

这会造成缓存雪崩风险。

比如：

```text
100 万个商品 key 都设置 EX 3600
```

一小时后大量 key 同时失效，请求会集中打到数据库。

更好的做法是 TTL 随机化：

```text
基础 TTL + 随机偏移
```

例如：

```text
30 分钟 + 0 到 10 分钟随机值
```

这样缓存过期时间会分散。

---

## 十三、哪些 key 必须设置 TTL

通常必须设置 TTL 的 key：

- 验证码。
- 登录 token。
- 临时授权码。
- 限流计数。
- 防重复提交标记。
- 空值缓存。
- 普通缓存数据。

通常不一定设置 TTL 的 key：

- 需要长期保留的计数器。
- 手动维护的配置缓存。
- 某些排行榜当前周期 key。

但即使不设置 TTL，也要明确原因。不要因为忘记了才不设置。

---

## 十四、项目级 key 规范示例

你可以在项目文档里写一份规范：

```text
1. Redis key 使用小写字母、数字、下划线和冒号。
2. 使用冒号分隔业务层级。
3. key 格式为：业务:对象:标识:用途。
4. 同一类 key 必须统一前缀。
5. 业务代码必须通过统一函数生成 key。
6. 临时数据必须设置 TTL。
7. 缓存 TTL 必须结合业务设置，并尽量加入随机偏移。
8. 禁止把大对象集合塞进单个 key。
9. 禁止在生产环境使用 KEYS * 排查。
10. 删除缓存前必须确认 key 前缀和影响范围。
```

短链接项目可以这样定义：

```text
shortlink:detail:{code}                短链接详情缓存，TTL 30 到 40 分钟
shortlink:not_found:{code}             空值缓存，TTL 1 到 3 分钟
shortlink:visit_count:{code}           访问计数，可长期保留并定时落库
shortlink:create_limit:user:{user_id}  创建限流，TTL 按窗口设置
```

这类规范越早写，后面越省心。

---

## 十五、反例分析

### 反例 1：key 太模糊

```text
cache:1001
```

问题：不知道是什么缓存，也不知道 1001 是用户 ID、文章 ID 还是商品 ID。

改成：

```text
article:detail:1001
```

### 反例 2：一个 key 塞太多数据

```text
user:all -> 所有用户 JSON
```

问题：value 过大，读取和删除都危险。

改成：

```text
user:profile:1001
user:profile:1002
```

### 反例 3：临时数据没有 TTL

```redis
SET sms:code:phone:13800138000 888888
```

问题：验证码不会自动失效。

改成：

```redis
SET sms:code:phone:13800138000 888888 EX 300
```

### 反例 4：缓存 TTL 太长

商品库存缓存 24 小时：

```redis
SET product:stock:1001 10 EX 86400
```

问题：库存变化频繁，用户可能看到严重旧数据。

库存这类数据要更谨慎，可能需要短 TTL、主动删除、数据库校验或其他一致性方案。

---

## 十六、本节练习

请完成下面练习：

1. 为用户资料设计一个 key 模板。
2. 为文章详情缓存设计一个 key 模板。
3. 为短信验证码设计一个 key 模板，并写出 Redis 命令。
4. 为登录 token 设计一个 key 模板，并写出 TTL 思路。
5. 为接口限流设计一个 key 模板。
6. 为短链接详情缓存设计一个 key 模板。
7. 判断 `cache:1001` 有什么问题。
8. 判断 `article:all:details` 可能有什么风险。
9. 给文章详情缓存设计一个带随机偏移的 TTL 策略。
10. 写一份你自己的 Redis key 命名规范，至少 6 条。

---

## 十七、本节小结

这一节你学习了第二阶段最核心的内容：key 设计和 TTL 规划。

你需要记住：

- key 要有清晰业务语义。
- 推荐使用冒号分隔层级。
- 同一类 key 必须风格一致。
- key 不要太短，也不要过度冗长。
- value 不要过大，避免 Big Key。
- 临时数据必须设置 TTL。
- 缓存 TTL 要结合业务，并尽量加入随机偏移。
- 项目里应该集中封装 key 生成函数。
- Redis 规范不是形式主义，它直接影响排查、性能和稳定性。

到这里，第二阶段的核心能力已经建立：你不仅会操作 Redis key，还能开始设计更清晰、更可靠的 Redis key。
