# Kafka学习

### 一、概念与理论

1、kafka的组件：

####  Producer（生产者）

- **作用**：把消息发送到 Kafka 的某个 Topic。
- **特点**：
  - 可选择发送到特定分区或由 Kafka 根据 key 自动分区。
  - 支持 **同步发送**（等 ACK）或 **异步发送**（回调处理）。
  - 可以设置 **ACK 策略**：
    - `acks=0`：不等待 Broker 确认 → 最快，但可能丢消息
    - `acks=1`：只等待 Leader 确认 → 性能好，稍有风险
    - `acks=all`：Leader 和 ISR 副本都确认 → 最安全

####  Consumer（消费者）

- **作用**：从 Kafka Topic 消费消息。
- **特点**：
  - 消费 **按分区顺序**，跨分区顺序无法保证。
  - 消费者通过 **Offset** 来记录消费进度(这个offset体现在kafka的分区里，即里面的数据被读到哪儿了)。
  - 消费模式：
    - 自动提交 Offset → 简单，但可能丢失或重复消息
    - 手动提交 Offset → 控制精确度，可实现幂等消费
  - **消费者组**：
    - 同组内消费者共享分区
    - 不同组可重复消费同一 Topic 消息

####  Broker & Cluster（节点与集群）

- **Broker**：单个 Kafka 服务节点，存储消息、处理请求
- **Cluster**：多个 Broker 组成，支持高可用
- **Leader / Follower**：
  - 每个分区有一个 Leader，处理读写请求
  - Follower 做副本同步，保证容错
- **分区策略**：
  - 根据 key 分配到特定分区
  - 无 key 随机或轮询分配

#### Kafka的消息存储和分区原理

##### 1.1 消息在 Kafka 中是怎么存储的？

​	Kafka 的 Topic 分为 **多个 Partition（分区）**

​	每个 Partition 是一个 **顺序写入的日志文件**，消息按 Offset 顺序追加

​	**消息持久化到磁盘**，默认不删除，按照 **保留策略**（时间或大小）清理

​	**读取**：消费者通过 Offset 拉取消息，Kafka **不会主动推送**

关于顺序读写：

​	首先 一条消息只会属于一个分区，如果有指定key做hash的话 kafka就会决定消息最后属于哪个分区。如果没有指定的话，那么消息则会以轮询的方式分配到分区。消息放到分区之后，在分区内是按照顺序存放的。每条消息进入到分区之后就会被赋予一个offset，offset是递增的。

​	消费者在消费某个分区的数据时，会记住自己消费到的offset，下次拉取消息时，会从对应的offset开始继续拉取消息。不过也可以支持任意位置消费，比如可以回溯到历史消息。

​	消息持久化：kafka按照topic配置的保留策略保留消息(时间或者大小),消费者消费完消息之后 不会立即删除，而且**允许多个消费者组独立消费**，比如 对于topic 'news'现在有两个分区：

``` 
Partition-0: Offset 0->A, 1->B, 2->C
Partition-1: Offset 0->X, 1->Y, 2->Z
```

消费者组1可以同时消费Partition-0和Partition-1。每个分区顺序保证，跨分区的话 顺序肯定就不保证了。

##### 1.2 Kafka的分区

分区就是为了提升吞吐量和提高拓展性，以及在一定程度上保证顺序。

首先是分区的策略

​	如果我们设置了hash规则，kafka会对key做hash，然后决定分区，这样同key的消息必定落到相同的分区，再加上之前的offset，又保证了消息的顺序了。这种场景比较适合比如订单、用户、设备id。

​	如果我们没有设置策略，就是上面已经提到的轮询，平均分布消息负载。

##### 1.3 Kafka的消费者组

​	对于多个kafka分区和消费者来说，同一个分区同时只能被一个消费者读取数据，但是一个消费者可以同时读取多个分区的数据。

为什么要如此设计？其实就是为了分区里消息的顺序性。

​	**Offset 并不是存在消费者本地**（虽然消费者会缓存一份进度），而是 Kafka **帮每个 Consumer Group 统一存储在内部 Topic `__consumer_offsets`** 里。**谁来读数据** → Kafka 根据这个 Group 的 offset 给你拉，从对应位置开始读。

所以同一个 Topic，**不同 Consumer Group 可以各自有自己的 offset**，互不干扰 → 同一条消息可以被多个系统消费。

只有保证了顺序，可回溯和高吞吐

##### 1.4 Leader / Follower机制

每个 **分区（Partition）** 在 Kafka 集群里都有 **一个 Leader** 和若干 **Follower 副本**：

```
Partition-X
 ├── Leader (处理读写请求)
 ├── Follower1 (同步Leader)
 └── Follower2 (同步Leader)
```

------

######  Leader 的角色

- **处理所有客户端读写请求**
  - Producer 写消息 → 写给 Leader
  - Consumer 拉取消息 → 从 Leader 读
- Leader 的 Offset 是分区的“标准序列”，Follower 会跟它同步

------

######  Follower 的角色

- **同步 Leader 消息**
  - 持续拉取 Leader 的数据
  - 保持 Offset 与 Leader 一致
- **不处理客户端请求**（默认）
- **容错备用**：
  - Leader 宕机 → Follower 中的 ISR（In-Sync Replica）选一个新的 Leader

######  ISR（In-Sync Replica）

- ISR = 当前与 Leader 保持同步的副本集合
- 只有在 ISR 内的 Follower 才能升级为 Leader
- 保证切换 Leader 后，数据不会丢失
- Kafka 默认：`min.insync.replicas=1~2`，写入必须有多少副本确认成功才算 ACK

------

###### 副本同步策略

Kafka 支持两种写入确认策略：

1. **acks=0**
   - Producer 不等待确认，极快，但不保证数据安全
2. **acks=1（默认）**
   - Leader 收到消息就返回 ACK
   - 如果 Leader 宕机、Follower 没同步完 → 数据可能丢失
3. **acks=all / -1**
   - 所有 ISR 副本确认收到消息才返回
   - 数据安全性最高 → 推荐生产环境使用

------

###### Leader 宕机处理流程

假设 Leader 宕机：

1. Kafka Controller（负责集群管理的 Broker）检测到 Leader 不可用
2. 从 ISR 中选择一个 Follower 升级为新 Leader
3. Producer / Consumer 自动重连到新 Leader
4. 消息继续写入 / 读取

📌 注意：如果 ISR 中没有同步完成的副本 → 可能会丢数据

- 所以通常推荐至少 2 个 Follower（副本数 >= 3）

### 二、简单安装与部署

##### 1、关于版本和镜像

##### 2、启动脚本和持久化配置

### 三、集群与高可用

### 四、spring中的使用

```pom
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
spring:
  kafka:
    bootstrap-servers: Kafka地址:9092     #集群地址
    consumer:
      group-id: test-group    #消费者组
      auto-offset-reset: earliest 
      enable-auto-commit: false  #手动提交offset
    producer:
      retries: 3
      acks: all    #消息持久化保证
      batch-size: 16384
      properties:
        linger.ms: 5
```



### 五、常见业务场景、问题及处理

### 六、线上问题排查