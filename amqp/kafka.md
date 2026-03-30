# Kafka

Apache Kafka 是分布式流平台，常用于**日志与事件流**、**消息缓冲**和**流处理**。下文整理特点、核心概念、副本与消费配置、语义与典型场景。

**结构导读**：特点与边界 → 适用场景 → 基础概念 → 副本与分区 → 配置要点 → 消费与积压 → 送达语义 → 应用场景 → 参考。

---

## 特点与边界

**常见优势**

1. **持久化**：日志分段落盘，可配置保留策略与压缩。
2. **高吞吐**：顺序写盘、批量发送/拉取，适合大流量。
3. **水平扩展**：通过增加 Broker、分区伸缩容量。
4. **多语言客户端**：生态成熟。
5. **Kafka Streams / 流处理**：与消息栈一体的流计算能力。
6. **安全**：SASL、TLS、ACL 等（依集群配置）。
7. **副本**：多副本提升可用性与数据冗余。
8. **消息压缩**：支持 gzip、snappy、lz4、zstd 等。

**与经典 MQ 的差异（开箱能力）**

Kafka **不强调**像 RabbitMQ 那样的延迟队列、优先级队列、内置消费重试 DLQ 模型；这些通常用**主题设计**（重试主题、死信主题）、**流处理**或**应用层**实现。

---

## 适用场景

- **日志 / 事件采集**、**消息总线**、**流式处理**、**削峰填谷**。
- **不适合**强依赖「分钟级定时触发」且要求中间件原生延迟队列的订单闭环（可改用定时任务 + 业务表或专用 MQ 能力）。

**概括**：天然适合**多生产者、多消费者**、**可回放**的日志型数据管道；持久化与高伸缩是其强项。

---

## 基础概念

| 概念 | 说明 |
|------|------|
| **Topic** | 逻辑上的消息类别；物理上由多个 **Partition** 组成。 |
| **Partition** | 有序日志；分区之间无序。是并行度与副本的单位。 |
| **Producer / Consumer** | 生产与消费记录；Consumer 通常属于某个 **Consumer Group**。 |
| **Offset** | 分区内的消费位移；由组协调器与 `__consumer_offsets`（或旧版 ZooKeeper）管理。 |

---

## 副本（Replication）

副本用于**容错**：同一分区的多个副本分布在不同 Broker 上。

- 默认副本因子常为 **1**（学习环境）；生产一般 **2～3**。副本过多会增加存储与复制带宽，需权衡。
- 副本角色：**Leader** 负责读写请求（生产者一般只写 Leader）；**Follower** 从 Leader **拉取**复制。

### AR / ISR / 掉队副本

- **AR（Assigned Replicas）**：分区分配到的全部副本。
- **ISR（In-Sync Replicas）**：与 Leader **保持同步**（在 `replica.lag.time.max.ms` 等阈值内）的副本集合。
- **未在 ISR 中的副本**：常称为**掉队副本**（因延迟过大被移出 ISR 等）。有资料用 **OSR** 等非官方缩写指代，与 ISR 相对；**不要**写成与官方文档混用的固定公式，只需理解：**Leader 选举优先在 ISR 内进行**。

### Leader 选举（要点）

Controller 在 Leader 失效时，通常从 **ISR** 中选出新 Leader。若 ISR 为空，是否允许从非 ISR 副本选举（**unclean leader election**）由 `unclean.leader.election.enable` 控制：打开可能丢数据，关闭可能短期不可用。

创建主题示例：

```bash
kafka-topics --create --topic test --replication-factor 2 --partitions 4 \
  --bootstrap-server 127.0.0.1:9092
# 4 个分区，副本因子 2
```

---

## 分区与消费者分配

### Consumer 分区分配策略

常见实现（通过客户端 `partition.assignment.strategy` 配置，名称随版本略有差异）：

- **Range**：按分区序号与消费者 id 排序后，按区间分配；**多 Topic** 时可能出现**分配不均**。
- **RoundRobin**：把所有 Topic 的分区与消费者轮询分配，相对更均匀。
- **Sticky / Cooperative-sticky**：尽量**保持分区粘性**；协作式再均衡时可减少不必要的撤销与停顿。

默认策略随 **Kafka 与客户端版本**而异，以当前客户端文档为准；不要死记「range + roundrobin 同时默认」。

> **约束**：**同一分区同一时刻只会有一个组内消费者读取**（同一 consumer group）。若消费者数 **大于** 分区数，多出来的消费者会空闲。

---

## 配置要点

### Broker / 系统级（摘录）

| 配置项 | 含义 |
|--------|------|
| `broker.id` | 集群内唯一整数；常与 IP 解耦，换 IP 可不换 id。 |
| `listeners` | 监听地址列表；`0.0.0.0` 表示绑定所有网卡。 |
| `zookeeper.connect` | **KRaft 模式前**的 ZK 连接串（新版本逐步以 KRaft 替代，部署时以实际模式为准）。 |
| `auto.create.topics.enable` | 是否允许自动建 Topic；生产常设为 **false**，避免误创建。 |
| `log.dirs` | 数据目录，可多路径；新分区会倾向落在负载较低的路径（实现细节以版本为准）。 |

### Topic 级（摘录）

- **`num.partitions`**：新 Topic 默认分区数。一般 **分区数 ≥ 峰值下消费者并行度**（并预留扩容空间）。分区策略见上文「分区与消费者分配」。
- **`message.max.bytes` / 相关 fetch 上限**：Broker、Producer、Consumer 侧**最大消息与拉取字节**需配套，否则大消息会被拒或截断。

### Producer（Go 示例片段）

```go
configMap.SetKey("request.timeout.ms", "30000") // 等待服务端响应的超时（毫秒）
```

`batch.size`、`linger.ms` 等用于批量与延迟折中；`batch.size` 单位为**字节**（默认约 16KB），调优时与 `linger.ms`、吞吐一起测。

### Consumer（Go 示例片段）

```go
configMap.SetKey("enable.auto.commit", "true")
configMap.SetKey("auto.commit.interval.ms", "5000")
```

> Confluent 等客户端的配置值多为**字符串**。

### 手动提交 Offset

关闭自动提交后，在业务合适时机调用 **`commitSync`（同步，可阻塞并重试）** 或 **`commitAsync`（异步，无内置重试）**。

常见做法：处理路径上**异步提交**降低延迟，**关闭/再均衡前**再 **同步提交** 一次，减少丢失已处理进度。

```go
configMap.SetKey("enable.auto.commit", "false")
```

**指定 Offset 消费**：各客户端 API 提供 `seek` / `assign` + 初始 offset 等能力，用于回放或跳过；具体见对应 SDK。

---

## 重复消费、漏消费与积压

### 重复消费

自动提交间隔内（如默认 5s）若**已处理数据但尚未 commit** 就崩溃，重启后会从**上次已提交 offset** 再拉，导致**重复**。业务侧应**幂等**（唯一键去重、状态机、数据库约束等）。

### 漏消费

若**先异步提交了 offset**，但消息还在内存中**未处理完**进程就被 kill，可能**已提交但未处理**，造成**漏消费**。

缓解思路：将「**处理结果**」与「**offset 记录**」放在**同一事务边界**（例如消费后写库与写 offset 同事务），或使用事务型 Producer/Exactly-once 能力（见下文），按业务复杂度选型。

### 积压与吞吐

1. **消费跟不上**：增加 **分区数** 并增加**同组消费者**（消费者数 ≤ 分区数才有意义）。
2. **单批处理慢**：适当增大 **`fetch.max.bytes`、每 poll 条数** 等（注意内存与处理延迟）；同时优化下游逻辑。

示例（需按集群与消息体大小压测调整）：

```go
configMap.SetKey("batch.size", "16384")   // 字节，按实际调优，非越大越好
configMap.SetKey("linger.ms", "5")
configMap.SetKey("fetch.max.bytes", "52428800")
```

---

## 消息送达语义

| 语义 | 含义 |
|------|------|
| **At most once** | 至多一次，可能丢、不重复。 |
| **At least once** | 至少一次，**可能重复**，一般不丢（视配置与故障模式）。 |
| **Exactly once** | 端到端「恰好一次」需在 **幂等 Producer + 事务** 等与下游配合；Kafka 提供相关构建块，**不是**所有场景开箱即得。 |

默认 Producer 在未开幂等时，常见为 **at least once** 倾向；云厂商文档也普遍强调消费侧按 **at least once** 做幂等。

### 消费失败怎么办

1. **重试**：注意阻塞与顺序，避免无限重试导致积压。
2. **死信 / 失败队列**：失败记录发到专用 Topic 或存储，配合告警与人工/定时修复。

---

## 应用场景（配图见原文路径）

### 1. 异步处理（注册发邮件/短信）

串行、并行与引入消息队列的对比见下图（响应时间主要变为写库 + 投递 Kafka）。

![用户注册串行化](https://iscod.github.io/images/kafka1.png)

![用户注册并行处理](https://iscod.github.io/images/kafka2.png)

![用户注册消息队列](https://iscod.github.io/images/kafka3.png)

### 2. 应用解耦（订单与库存等）

订单系统落库后发事件到 Kafka，库存、积分、物流等各自订阅；下游短时不可用**不阻塞**订单主路径（仍需设计补偿与最终一致）。

![传统模式用户下单](https://iscod.github.io/images/kafka4.png)

![引入消息队列用户下单](https://iscod.github.io/images/kafka5.png)

### 3. 流量削峰（秒杀等）

请求先入队，后端按能力消费；队列过长时可配合**限流、降级、丢弃或排队页**。

![流量削峰](https://iscod.github.io/images/kafka6.png)

### 4. 日志同步

采集端 → Kafka → 实时/离线分析（如 Flink、ELK 等）。

![日志同步](https://iscod.github.io/images/kafka7.png)

### 5. 活动跟踪

将用户行为事件写入 Topic，供推荐、风控、运营分析等订阅消费。

### 6. 流处理

与 **Kafka Streams** 或 **Flink** 等结合，做窗口聚合、连接流、实时指标。

---

## 参考

- [Kafka 相关（腾讯云开发者）](https://cloud.tencent.com/developer/article/1974648)
- [Kafka 场景说明（华为云）](https://support.huaweicloud.com/productdesc-kafka/kafka-scenarios.html)
- [Kafka、RabbitMQ、RocketMQ 差异（华为云）](https://support.huaweicloud.com/productdesc-kafka/kafka_pd_0003.html)
