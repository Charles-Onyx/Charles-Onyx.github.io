---
layout: post
title: "从零构建物联网数据平台：高并发传感器数据接入实践"
date: 2026-06-12 10:00:00 +0800
categories: ["后端开发", "架构设计"]
tags: [Java, Spring Boot, Redis, RocketMQ, 高并发, IoT]
---

在智慧农业场景中，一套物联网平台需要同时接入成千上万个传感器设备，每个设备以秒级频率上报温度、湿度、光照、土壤 pH 值等监测数据。如何设计一套能够**稳定处理海量并发写入**的数据接入层，是后端架构师面临的第一道考题。

## 一、业务背景与挑战

以我参与开发的智慧农业大数据管理平台为例，系统需要面向现代大型农场提供设备集控与环境监测能力。实际生产环境中，一个中型农场部署的传感器节点可达 5000+，单节点每秒上报 1-2 条数据，峰值 QPS 轻松突破 10000。

**核心挑战**：
1. **写入洪峰**：设备上报集中在整点前后，存在明显的流量尖峰
2. **数据完整性**：农业数据具备时序特征，丢失一条可能导致分析偏差
3. **后端稳定性**：数据库直接承载写入压力极易造成连接池耗尽

## 二、架构设计：三层削峰写入模型

解决高并发写入的核心思路是——**不直接写数据库**。我们设计了经典的"生产者-缓冲-消费者"模型：

```
传感器 → 负载均衡 → 接收网关 → Redis 队列 → RocketMQ → 批量消费 → MySQL
```

### 2.1 接收网关层

接收网关基于 Netty 构建，负责维持与传感器设备的长连接：

```java
// 网关核心：基于 Netty 的异步非阻塞接收
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(Runtime.getRuntime().availableProcessors() * 2);

ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) {
            ch.pipeline()
                .addLast(new IdleStateHandler(60, 0, 0))
                .addLast(new SensorDataDecoder())    // 自定义协议解码
                .addLast(new DataProcessHandler());   // 业务处理
        }
    })
    .option(ChannelOption.SO_BACKLOG, 1024)
    .childOption(ChannelOption.SO_KEEPALIVE, true)
    .childOption(ChannelOption.TCP_NODELAY, true);
```

关键优化点：
- 使用 `SO_BACKLOG = 1024` 应对瞬时并发连接
- TCP_NODELAY 禁用了 Nagle 算法，减少网络延迟
- 工作线程数设为 CPU 核心数的 2 倍，避免上下文频繁切换

### 2.2 Redis 缓存缓冲层

每条数据先进入 **Redis List**，而非直接落入数据库：

```java
// 接收网关 -> Redis List
public void cacheSensorData(SensorData data) {
    // 按设备 ID 分片，避免单队列过大
    String queueKey = "sensor:queue:" + (data.getDeviceId() % SHARD_COUNT);
    redisTemplate.opsForList()
        .rightPush(queueKey, JSON.toJSONString(data));
}
```

为什么先走 Redis：
- Redis 单机可达 10万+ QPS 的写入能力，天然抗住洪峰
- List 的 LPUSH/RPUSH 操作复杂度 O(1)，写入几乎无延迟
- 削峰填谷：消费者按自身节奏从队列拉取，数据库不会被打死

### 2.3 RocketMQ 消息队列

Redis 作为一级缓冲可以暂存千万级数据，但我们还需要**可靠的消息投递**。RocketMQ 在这里扮演了二级队列的角色：

```java
@Component
public class SensorDataProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public void sendToMQ(List<SensorData> batch) {
        Message<List<SensorData>> msg = MessageBuilder
            .withPayload(batch)
            .build();
        // 同步发送，确保到达
        SendResult result = rocketMQTemplate.syncSend(
            "sensor-data-topic", msg, 3000);
        if (result.getSendStatus() != SendStatus.SEND_OK) {
            log.error("MQ send failed: {}", result);
            // 补偿：写入本地文件或备用队列
        }
    }
}
```

选择 RocketMQ 而非 Kafka 的原因：
- 支持**事务消息**，保证数据最终一致性
- 消息轨迹功能便于排查丢数据问题
- 对 Java 生态的原生支持更好

### 2.4 批量写入 MySQL

消费端将数据攒批后批量写入：

```java
@Component
public class SensorDataConsumer {

    private static final int BATCH_SIZE = 500;
    private final List<SensorData> batch = new ArrayList<>();

    @Transactional
    public void consume(String message) {
        List<SensorData> dataList = parseMessage(message);
        synchronized (batch) {
            batch.addAll(dataList);
            if (batch.size() >= BATCH_SIZE) {
                flushBatch();
            }
        }
    }

    private void flushBatch() {
        // 使用 JDBC batch 替代逐条 INSERT
        jdbcTemplate.batchUpdate(
            "INSERT INTO sensor_data (device_id, type, value, ts) VALUES (?, ?, ?, ?)",
            batch,
            BATCH_SIZE,
            (ps, data) -> {
                ps.setLong(1, data.getDeviceId());
                ps.setString(2, data.getType());
                ps.setDouble(3, data.getValue());
                ps.setTimestamp(4, data.getTimestamp());
            });
        batch.clear();
    }
}
```

**性能对比**：500 条一批写入，相比逐条 INSERT 性能提升约 20 倍。

## 三、压测效果

使用 JMeter 模拟 2000 个设备并发上报，每个设备每秒发送 2 条数据：

| 指标 | 接入 Redis | 直接写 MySQL |
|------|-----------|-------------|
| 峰值吞吐 | **48500 TPS** | 3200 TPS |
| P99 延迟 | **12ms** | 890ms |
| 数据库 CPU | 15% | **92%** |

## 四、踩坑记录

1. **Redis 内存飙升**：未设置 TTL 的 List 持续增长，差点撑爆内存。解决方案：设置队列上限 + 定时清理策略。

2. **Netty 内存泄漏**：ByteBuf 未正确释放导致堆外内存泄漏。通过 `Netty.leakDetectionLevel = advanced` 定位并修复。

3. **批量写入死锁**：多个消费者线程同时批量写入同一张表时出现死锁。优化方案：按 device_id 哈希分表，每个线程固定写一张子表。

## 五、总结

高并发数据接入的核心思想是**分层缓冲、逐级削峰**——每一层只做自己最擅长的事：Netty 擅长连接管理，Redis 擅长快速写入，RocketMQ 擅长可靠投递，MySQL 擅长结构化存储。让合适的工具做合适的事，系统自然稳定。
