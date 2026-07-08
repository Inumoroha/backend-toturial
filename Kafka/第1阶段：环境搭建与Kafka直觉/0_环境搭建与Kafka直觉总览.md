# 第 1 阶段：环境搭建与 Kafka 直觉总览

这一阶段的目标不是马上写 Go 代码，而是先建立 Kafka 的基本直觉：它为什么存在、本地环境怎么启动、怎么用命令行收发消息、怎么观察 topic 和 consumer group。

如果你以前学过 PostgreSQL，可以把这一阶段理解成 Kafka 版本的“启动数据库、连接数据库、创建第一张表”。区别是 Kafka 的核心对象不是 table 和 row，而是 topic、partition、message、offset 和 consumer group。

---

## 一、本阶段你会学什么

本阶段包含以下课程：

1. [使用 Docker 启动 Kafka](1_使用Docker启动Kafka.md)
2. [使用 Kafka 命令行工具](2_使用Kafka命令行工具.md)
3. [理解消息队列、事件流与 Kafka 的定位](3_理解消息队列事件流与Kafka定位.md)
4. [理解 Topic、Partition、Message、Offset 的基本直觉](4_理解Topic_Partition_Message_Offset.md)
5. [阶段综合实践：从零完成一次消息收发](5_综合实践_从零完成一次消息收发.md)

---

## 二、本阶段学习目标

完成这一阶段后，你应该能做到：

- 使用 Docker Compose 启动 Kafka。
- 知道 Kafka 为什么要配置 listener。
- 使用命令行创建 topic。
- 使用命令行 producer 发送消息。
- 使用命令行 consumer 消费消息。
- 查看 consumer group 的 offset 和 lag。
- 用自己的话解释 topic、partition、message、offset。
- 知道 Kafka 与普通消息队列的区别。

---

## 三、本阶段不急着学什么

这些内容后面会详细讲，本阶段只需要有印象：

- 多 broker 集群。
- 副本和 ISR。
- producer 幂等。
- Kafka 事务。
- Go 客户端封装。
- retry topic 和 DLQ。
- 性能压测。

学习 Kafka 很容易一开始就陷进参数海洋。初学阶段最重要的是：先让消息从 producer 进入 Kafka，再从 consumer 读出来。

---

## 四、建议学习方式

每学一节，都按下面顺序做：

```text
先读概念。
再复制命令运行。
观察终端输出。
记录和预期是否一致。
如果不一致，先看日志和端口。
最后用自己的话写 5 行总结。
```

不要只看文档，不敲命令。Kafka 的很多概念，比如 offset、consumer group、lag，只有在终端里看过变化，才会真的理解。

---

## 五、本阶段最终验收

你需要能独立完成：

1. 启动 Kafka。
2. 创建 `quickstart.orders` topic。
3. 发送 5 条订单消息。
4. 使用一个 consumer group 消费消息。
5. 查看这个 group 的 lag。
6. 停止 consumer 后继续发送消息。
7. 再次查看 lag 增长。
8. 重启 consumer，观察 lag 下降。

如果能做到这些，就可以进入第 2 阶段。

---

## 六、本阶段最终产出

完成第 1 阶段后，建议你在本地留下这些东西：

```text
docker-compose.yml
commands.md
quickstart-notes.md
```

`commands.md` 里至少记录：

```text
启动 Kafka 的命令。
创建 topic 的命令。
发送消息的命令。
消费消息的命令。
查看 consumer group lag 的命令。
停止和清理环境的命令。
```

这些命令后面每个阶段都会反复用到。不要每次都重新搜索。

---

## 七、最小学习闭环

本阶段最重要的闭环是：

```text
我能启动 Kafka
-> 我能创建 topic
-> 我能发送消息
-> 我能消费消息
-> 我能看到 offset 和 lag 变化
```

如果你只完成了“启动成功”，还不算真正学完本阶段。Kafka 的直觉来自消息流动，而不是容器状态显示 running。

---

## 八、常见卡点

### 1. Kafka 容器启动了，但客户端连不上

优先检查 listener 配置。Docker 内部访问和宿主机访问使用的地址经常不同。

### 2. 命令行工具找不到

如果使用 Docker，可以进入 Kafka 容器执行命令，也可以使用本机安装的 Kafka CLI。关键是 `--bootstrap-server` 地址要正确。

### 3. consumer 没读到旧消息

检查是否加了 `--from-beginning`，以及 consumer group 是否已经提交过 offset。

### 4. lag 看不懂

记住：

```text
lag = topic 最新 offset - group 已提交 offset
```

consumer 停止后继续生产消息，lag 应该增长；consumer 恢复并处理消息后，lag 应该下降。

---

## 九、进入下一阶段前的自测

你应该能不看教程说出：

1. topic 是什么？
2. message 是什么？
3. partition 是什么？
4. offset 是什么？
5. consumer group 是什么？
6. 为什么停止 consumer 后 producer 仍然可以写消息？
7. 为什么 consumer 恢复后还能继续读之前没处理的消息？

如果这些还模糊，建议再重复做一遍综合实践。

---

## 十、本阶段推荐记录模板

建议你在学习时建立一个 `quickstart-notes.md`，记录每次实验：

```markdown
# Kafka 第 1 阶段实验记录

## 环境

- Docker 版本：
- Kafka 镜像：
- bootstrap server：

## Topic

- topic 名称：
- partitions：

## Producer 实验

- 发送命令：
- 发送消息数量：
- 是否成功：

## Consumer 实验

- group id：
- 是否使用 --from-beginning：
- 观察到的消息：

## Lag 实验

- 停止 consumer 后 lag：
- 重启 consumer 后 lag：

## 遇到的问题
```

这份笔记会在后面排查 offset、consumer group、lag 时反复用到。

---

## 十一、阶段学习节奏

建议用 2 到 3 天完成本阶段：

| 天数 | 任务 |
| --- | --- |
| 第 1 天 | 启动 Kafka，理解 listener 和 bootstrap server |
| 第 2 天 | 熟悉 topic、producer、consumer 命令 |
| 第 3 天 | 做综合实践，观察 consumer group lag |

如果 Docker、端口、网络配置卡住，不要急着进入下一阶段。Kafka 后面所有实验都依赖这个环境。

---

## 十二、和 PostgreSQL 学习的类比

可以这样类比：

| PostgreSQL | Kafka |
| --- | --- |
| 启动数据库 | 启动 broker |
| 创建 table | 创建 topic |
| insert row | produce message |
| select row | consume message |
| primary key | message key |
| WAL | partition log |

类比只是帮助入门，不要完全等同。Kafka 不是数据库，它更像一条可持久化、可回放的事件日志。

---

## 十三、本阶段完成标准

你可以进入下一阶段的标志是：

- 不看教程也能启动 Kafka。
- 能独立创建 topic。
- 能解释为什么 consumer group 需要 group id。
- 能制造 lag 增长和下降。
- 能知道消息读过后为什么还在 topic 里。
- 遇到连接失败时知道先检查 listener、端口和 bootstrap server。

这时再学 broker、partition、replica、offset 的内部机制，会更容易理解。

---

## 十四、推荐复盘问题

完成综合实践后，建议写一段复盘：

```text
我启动 Kafka 时遇到的问题是什么？
我创建 topic 时指定了哪些参数？
producer 发送消息后，consumer 是如何读到的？
consumer group 的 lag 在什么时候增长？
lag 又是在什么时候下降？
```

复盘不是写给别人看的，而是帮助你把“跑过命令”变成“理解链路”。

---

## 十五、下一阶段预告

第 2 阶段会解释本阶段看到的现象背后的原因：

- 为什么 topic 可以有多个 partition。
- 为什么 offset 是数字。
- 为什么 consumer group 能记录消费进度。
- 为什么消息读过以后不会马上删除。
- 为什么 broker 可以通过副本提高可用性。

带着这些问题进入第 2 阶段，会比直接背概念更有效。

---

## 十六、阶段结束小作业

请独立完成一次“不看教程”的演示：

```text
1. 清理旧容器。
2. 启动 Kafka。
3. 创建 topic。
4. 发送 3 条消息。
5. 使用新 group 消费。
6. 查看 lag。
7. 停止 consumer 后再发送 3 条。
8. 解释 lag 为什么变化。
```

如果演示过程中卡住，把卡住的位置写进笔记。这个卡点就是下一轮复习重点。

---

## 十七、学习产物检查

本阶段结束时，应该留下：

- 一份可用的 Docker Compose 配置。
- 一份 Kafka CLI 常用命令笔记。
- 一次 producer/consumer 实验记录。
- 一次 lag 增长和下降实验记录。
- 一份常见连接错误排查笔记。

这些材料会让你后面每个阶段都少花很多时间在环境问题上。

---

## 十八、面试表达

可以这样描述 Kafka 的第一直觉：

```text
Kafka 可以理解为持久化的事件日志。producer 把消息写入 topic，topic 被拆成 partition；consumer group 记录自己消费到的 offset。
消息不会因为某个 consumer 读过就立即删除，而是按 retention 保留，所以落后的 consumer 可以在保留期内继续追赶。
```

---

## 十九、推荐学习时间安排

| 天数 | 任务 | 产出 |
| --- | --- | --- |
| 第 1 天 | Docker 启动 Kafka | 可用的本地环境 |
| 第 2 天 | Kafka CLI | topic、producer、consumer 命令 |
| 第 3 天 | 消息收发综合实践 | lag 观察记录 |

如果环境搭建花了更久，也很正常。消息系统学习里，环境和命令本身就是基础能力。

---

## 二十、命令速查清单

本阶段结束时，至少能默写这些命令的用途：

```text
kafka-topics.sh --create
kafka-topics.sh --list
kafka-topics.sh --describe
kafka-console-producer.sh
kafka-console-consumer.sh
kafka-consumer-groups.sh --describe
```

具体参数可以查文档，但命令解决什么问题要记住。

---

## 二十一、最终检查

进入下一阶段前，重新打开一个新终端，完整跑一遍消息收发。

如果离开当前终端或重启电脑后仍然能独立跑通，说明你真正掌握了环境，而不是只跟着历史命令走了一遍。

---

## 二十二、下一步连接

第 1 阶段负责“看见消息流动”，第 2 阶段负责解释“消息为什么这样流动”。如果你已经能稳定跑通本阶段实验，就可以继续研究 broker、partition、replica 和 offset 的内部机制。

---

## 二十三、完成提示

不要删除本阶段命令笔记。后续每一阶段都会继续用到创建 topic、发送消息、消费消息和查看 lag 这些基础命令。

这些基础命令越熟，后面排查 Kafka 问题时越从容。

把它们当成 Kafka 学习的基本功。

---

## 二十四、复习频率

建议每完成两个阶段，就回到本阶段重新跑一次：

```text
create topic -> produce -> consume -> describe group
```

Kafka 后续概念再复杂，最终都会回到这条最基础的消息流。

---

## 二十五、故障演练

本阶段最后可以做一次小演练：

```text
把 bootstrap server 写错。
观察 producer 报错。
再改回正确地址。
确认消息可以正常发送。
```

这能帮助你熟悉最常见的连接问题。

---

## 二十六、最终产物

本阶段完成后，至少留下：

```text
docker-compose.yml
kafka-commands.md
lag-observation.md
```

这些文件会成为后续阶段反复使用的基础资料。

---

## 二十三、按 PostgreSQL 教程方式自查

学完本阶段后，不要只确认“命令跑通了”，而要留下可以复盘的证据。

建议你逐项检查：

```text
能否说清 broker、topic、partition、message、offset 的关系。
能否独立启动 Kafka，并用命令行完成一次生产和消费。
能否解释同一个 topic 为什么可以有多个 partition。
能否用 consumer group 工具看到当前 offset 和 lag。
能否把一次消息收发写成清晰的复现实验步骤。
```

如果其中任何一项说不清，就回到对应小节重新做一遍。Kafka 的后续内容都建立在这几个直觉之上。
