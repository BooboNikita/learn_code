# Kafka 核心概念与应用

> 来源：[深入解析 Kafka：从核心概念到实战应用](https://zhuanlan.zhihu.com/p/1955307495632397793)
> 一句话总结：Kafka 是高吞吐、低延迟的分布式消息流平台，通过零拷贝/顺序写/批量压缩实现百万级 QPS，广泛应用于日志聚合、微服务解耦、实时流处理。

## 一、核心架构

### 1.1 四大核心组件

| 组件 | 职责 | 说明 |
| --- | --- | --- |
| Broker | 接收、存储、转发消息 | 独立 Kafka 服务器，横向扩展组成集群 |
| Topic | 消息的逻辑分类单元 | 类似"数据管道"，生产者发布、消费者订阅 |
| Partition | Topic 的物理分片 | 有序、不可变日志序列，支持并行处理和水平扩展 |
| Producer / Consumer | 生产者 / 消费者 | Producer 发布消息到指定分区，Consumer 拉取模式读取 |

### 1.2 关键特性

**消费者组（Consumer Group）：**

- 多个消费者组成逻辑分组，同一分区消息仅被组内**一个**消费者处理
- 实现负载均衡与容错：组内某消费者故障，其他消费者自动接管

**元数据管理演进：**

| 模式 | 说明 |
| --- | --- |
| ZooKeeper 模式（早期） | 依赖外部 ZK 管理元数据、协调 Broker、选举 Leader |
| KRaft 模式（新版本） | 自研 Raft 共识协议，摆脱 ZK 依赖，简化架构、提升性能 |

---

## 二、高性能核心技术

### 2.1 性能优化三件套

| 技术 | 原理 | 效果 |
| --- | --- | --- |
| **零拷贝（Zero Copy）** | 利用 Linux `sendfile`，数据从页缓存直接传到网络套接字，跳过用户态拷贝 | 减少 CPU 开销和传输延迟 |
| **顺序写 + 页缓存** | 消息按顺序追加写入磁盘分区，结合 OS 页缓存，读操作直接命中缓存 | 顺序写吞吐量远高于随机写，减少磁盘 I/O |
| **批量处理 + 压缩** | 生产者将多条消息打包发送，支持 Snappy/Gzip 压缩 | 单机吞吐量轻松达百万级消息/秒 |

### 2.2 持久化存储

- 消息默认持久化到磁盘，通过**分段日志（Log Segment）+ 索引文件**实现高效读写
- 支持 TB 级数据存储，满足大数据时代持久化和容量需求

---

## 三、分布式容错与高可用

### 3.1 副本机制

- 每个分区支持多副本：1 个 **Leader**（处理读写）+ 多个 **Follower**（数据同步）
- **ISR（In-Sync Replicas）** 动态维护同步副本集合，只有与 Leader 保持同步的 Follower 才被纳入 ISR
- N-1 节点故障时，只要 ISR 中还有副本存活，数据不丢失

### 3.2 自动故障转移

- Leader 宕机时，从 ISR 的 Follower 中快速选举新 Leader
- 选举过程对消费者完全无感知，保证服务连续性

---

## 四、应用场景

### 4.1 典型场景对比

| 场景 | Kafka 角色 | 关键需求 |
| --- | --- | --- |
| 日志聚合 | 统一收集各节点日志 → ELK / HDFS | 高吞吐、持久化存储 |
| 微服务解耦 | 服务间异步通信，独立扩展 | 低延迟、消费组负载均衡 |
| 用户活动跟踪 | 实时采集行为数据 → 推荐/监控/数仓 | 实时性、多下游分发 |
| 流处理管道 | Kafka → Kafka Streams/Flink/Spark Streaming | 端到端实时流处理 |
| 金融交易 | 交易日志有序存储、实时对账 | 强持久化、副本一致性 |
| 物联网 IoT | 百万设备并发接入、实时响应 | 高并发、低延迟 |

### 4.2 与其他 MQ 对比

| 特性 | Kafka | RocketMQ | RabbitMQ |
| --- | --- | --- | --- |
| 吞吐量 | 极高（100W+/s） | 极高（10W+/s） | 中等（万级/s） |
| 延迟 | 毫秒级 | 毫秒级 | 微秒级 |
| 持久化 | 默认磁盘持久化 | 默认磁盘持久化 | 可配 |
| 顺序消息 | 分区有序 | 队列级严格有序 | 不支持 |
| 事务消息 | 不支持 | 支持 | 不支持 |
| 消息回溯 | 支持 | 支持 | 不支持 |
| 适用场景 | 日志、大数据、流处理 | 电商、金融 | 企业集成 |

---

## 五、快速入门

### 5.1 本地部署关键配置

```properties
# server.properties
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=/data/kafka-logs
# ZK 模式
zookeeper.connect=localhost:2181
# KRaft 模式（新版本）
controller.quorum.voters=...
```

**启动顺序：** ZK 模式先启动 ZooKeeper → 再启动 Kafka Broker；KRaft 模式直接启动 Broker。

### 5.2 Java 生产者示例

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
for (int i = 0; i < 10; i++) {
    producer.send(new ProducerRecord<>("test", "key" + i, "value" + i));
}
producer.close();
```

### 5.3 Java 消费者示例

```java
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList("test"));
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("Offset=%d, Key=%s, Value=%s%n",
            record.offset(), record.key(), record.value());
    }
}
```

---

## 六、未来演进

| 方向 | 说明 |
| --- | --- |
| KRaft 模式 | 摆脱 ZooKeeper 依赖，自管理元数据，简化运维 |
| Kafka Connect | 异构系统数据同步，连接各种数据源和目标 |
| Kafka Streams | 轻量级流处理 SDK，无需外部依赖 |
| 云原生适配 | 面向 Kubernetes / Serverless 轻量化演进 |

---

## 复习清单

1. **Kafka 四大核心组件？** Broker（服务器）、Topic（逻辑分类）、Partition（物理分片）、Producer/Consumer（生产/消费者）。
2. **消费者组的核心规则？** 同一分区消息仅被组内一个消费者处理，实现负载均衡和容错。
3. **ZooKeeper vs KRaft？** 早期依赖 ZK 管理元数据；新版本用 KRaft（Raft 共识协议）自管理，摆脱外部依赖。
4. **零拷贝技术原理？** 利用 `sendfile` 系统调用，数据从页缓存直接传到网络套接字，跳过用户态-内核态的冗余拷贝。
5. **Kafka 高吞吐的三个原因？** 零拷贝、顺序写+页缓存、批量处理+压缩。
6. **ISR 是什么？** In-Sync Replicas，与 Leader 保持同步的 Follower 副本集合，用于选举和容错。
7. **Leader 宕机怎么处理？** 从 ISR 的 Follower 中快速选举新 Leader，对消费者无感知。
8. **Kafka vs RocketMQ 核心区别？** Kafka 吞吐更高、不支持事务消息；RocketMQ 支持事务消息、队列级严格顺序。
9. **Kafka 典型应用场景？** 日志聚合、微服务解耦、用户活动跟踪、流处理管道、金融交易、物联网。
10. **消息持久化机制？** 分段日志（Log Segment）+ 索引文件，默认持久化到磁盘，支持 TB 级存储。
11. **消费者拉取模式优势？** 消费者主动控制拉取速率，避免被推送淹没；支持按需重放历史消息。
