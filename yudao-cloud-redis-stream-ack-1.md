# yudao-cloud Redis Stream 消息从 send 到 onMessage 消费完整路径分析

## 一、核心类协作关系

### 1.1 参与角色

| 类名 | 职责 | 位置 |
|------|------|------|
| `RedisMQTemplate` | 消息发送模板，统一封装 pub/sub 和 Stream 两种发送方式 | `yudao-spring-boot-starter-mq/core/RedisMQTemplate.java` |
| `AbstractRedisStreamMessageListener` | Redis Stream 消息监听器抽象基类，实现集群消费 | `yudao-spring-boot-starter-mq/core/stream/AbstractRedisStreamMessageListener.java` |
| `AbstractRedisChannelMessageListener` | Redis Pub/Sub 消息监听器抽象基类，实现广播消费 | `yudao-spring-boot-starter-mq/core/pubsub/AbstractRedisChannelMessageListener.java` |
| `RedisMessageInterceptor` | 消息拦截器接口，用于扩展消息发送/消费前后的逻辑 | `yudao-spring-boot-starter-mq/core/interceptor/RedisMessageInterceptor.java` |
| `AbstractRedisMessage` | Redis 消息抽象基类，提供 headers 字段存储扩展信息 | `yudao-spring-boot-starter-mq/core/message/AbstractRedisMessage.java` |
| `TenantRedisMessageInterceptor` | 多租户上下文拦截器实现 | `yudao-spring-boot-starter-biz-tenant/core/mq/redis/TenantRedisMessageInterceptor.java` |
| `RedisPendingMessageResendJob` | 定时处理 pending message 的重发任务 | `yudao-spring-boot-starter-mq/core/job/RedisPendingMessageResendJob.java` |

### 1.2 整体消息流转架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          消息发送端 (Producer)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Business Code (业务代码)                                                   │
│         │                                                                   │
│         ▼                                                                   │
│  RedisMQTemplate.send(T message)  ──── Stream 或 Pub/Sub                    │
│         │                                                                   │
│         ├─► sendMessageBefore(message)  [拦截器正序执行]                     │
│         │                                                                   │
│         ├─► 实际发送 (Redis Stream: XADD / Pub/Sub: PUBLISH)                │
│         │                                                                   │
│         └─► sendMessageAfter(message)   [拦截器倒序执行]                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼  Redis 消息持久化
┌─────────────────────────────────────────────────────────────────────────────┐
│                          消息消费端 (Consumer)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  StreamMessageListenerContainer / RedisMessageListenerContainer             │
│         │                                                                   │
│         ▼                                                                   │
│  AbstractRedisStreamMessageListener.onMessage(record)                      │
│  AbstractRedisChannelMessageListener.onMessage(message, bytes)              │
│         │                                                                   │
│         ├─► JsonUtils.parseObject()  反序列化消息                            │
│         │                                                                   │
│         ├─► consumeMessageBefore(message)  [拦截器正序执行]                  │
│         │                                                                   │
│         ├─► this.onMessage(message)  业务消费逻辑 (可能抛异常)              │
│         │                                                                   │
│         ├─► acknowledge(group, message)  Stream 才有 ack                   │
│         │                                                                   │
│         └─► consumeMessageAfter(message)   [拦截器倒序执行，finally 块]      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 二、问题详细解答

### 2.1 拦截器方法的执行顺序

#### 发送端拦截器顺序

**代码位置：** `RedisMQTemplate.java:75-85`

```java
private void sendMessageBefore(AbstractRedisMessage message) {
    // 正序
    interceptors.forEach(interceptor -> interceptor.sendMessageBefore(message));
}

private void sendMessageAfter(AbstractRedisMessage message) {
    // 倒序
    for (int i = interceptors.size() - 1; i >= 0; i--) {
        interceptors.get(i).sendMessageAfter(message);
    }
}
```

**结论：**
- `sendMessageBefore`：**正序**执行（添加顺序）
- `sendMessageAfter`：**倒序**执行（从后往前）

#### 消费端拦截器顺序

**代码位置：** `AbstractRedisStreamMessageListener.java:103-117` 和 `AbstractRedisChannelMessageListener.java:87-101`

```java
private void consumeMessageBefore(AbstractRedisMessage message) {
    List<RedisMessageInterceptor> interceptors = redisMQTemplate.getInterceptors();
    // 正序
    interceptors.forEach(interceptor -> interceptor.consumeMessageBefore(message));
}

private void consumeMessageAfter(AbstractRedisMessage message) {
    List<RedisMessageInterceptor> interceptors = redisMQTemplate.getInterceptors();
    // 倒序
    for (int i = interceptors.size() - 1; i >= 0; i--) {
        interceptors.get(i).consumeMessageAfter(message);
    }
}
```

**结论：**
- `consumeMessageBefore`：**正序**执行
- `consumeMessageAfter`：**倒序**执行（在 finally 块中执行）

#### 完整时序（假设有 2 个拦截器 A、B，按顺序添加）

```
发送端：
  ┌─► sendMessageBefore(A)
  │
  ├─► sendMessageBefore(B)
  │
  ├─► 实际发送消息到 Redis
  │
  ├─► sendMessageAfter(B)   (倒序，B 先)
  │
  └─► sendMessageAfter(A)

消费端：
  ┌─► consumeMessageBefore(A)
  │
  ├─► consumeMessageBefore(B)
  │
  ├─► 业务 onMessage() 执行
  │
  ├─► (Stream 才有) acknowledge() 执行
  │
  ├─► consumeMessageAfter(B)  (倒序，finally 块)
  │
  └─► consumeMessageAfter(A)
```

### 2.2 onMessage 抛异常时的 ack 和 consumeMessageAfter 执行

#### Stream 消费异常流程

**代码位置：** `AbstractRedisStreamMessageListener.java:63-80`

```java
@Override
public void onMessage(ObjectRecord<String, String> message) {
    T messageObj = JsonUtils.parseObject(message.getValue(), messageType);
    try {
        consumeMessageBefore(messageObj);
        // 消费消息
        this.onMessage(messageObj);  // <-- 如果这里抛异常
        // ack 消息消费完成
        redisMQTemplate.getRedisTemplate().opsForStream().acknowledge(group, message);
        // ...
    } finally {
        consumeMessageAfter(messageObj);  // <-- finally 块
    }
}
```

#### Pub/Sub 消费异常流程

**代码位置：** `AbstractRedisChannelMessageListener.java:55-64`

```java
@Override
public final void onMessage(Message message, byte[] bytes) {
    T messageObj = JsonUtils.parseObject(message.getBody(), messageType);
    try {
        consumeMessageBefore(messageObj);
        this.onMessage(messageObj);  // <-- 如果这里抛异常
    } finally {
        consumeMessageAfter(messageObj);  // <-- finally 块
    }
}
```

#### 结论

| 场景 | ack 是否执行 | consumeMessageAfter 是否执行 |
|------|-------------|-----------------------------|
| **Stream onMessage 抛异常** | **不执行**（ack 在 onMessage 之后） | **执行**（在 finally 块中） |
| **Stream consumeMessageBefore 抛异常** | **不执行** | **执行**（在 finally 块中） |
| **Pub/Sub onMessage 抛异常** | 不适用（Pub/Sub 无 ack） | **执行**（在 finally 块中） |

#### 代码中的 TODO 注释

在 `AbstractRedisStreamMessageListener.java:72-76` 有如下 TODO：

```java
// TODO 芋艿：需要额外考虑以下几个点：
// 1. 处理异常的情况
// 2. 发送日志；以及事务的结合
// 3. 消费日志；以及通用的幂等性
// 4. 消费失败的重试，https://zhuanlan.zhihu.com/p/60501638
```

说明当前实现中：
- 异常会导致消息不被 ack，成为 pending message
- 但 `consumeMessageAfter` 会在 finally 中执行，用于清理上下文

### 2.3 泛型消息类型的 TypeUtil 解析

#### 解析原理

**代码位置：** `AbstractRedisStreamMessageListener.java:94-101`

```java
@SuppressWarnings("unchecked")
private Class<T> getMessageClass() {
    Type type = TypeUtil.getTypeArgument(getClass(), 0);
    if (type == null) {
        throw new IllegalStateException(String.format("类型(%s) 需要设置消息类型", getClass().getName()));
    }
    return (Class<T>) type;
}
```

**代码位置：** `AbstractRedisChannelMessageListener.java:78-85` 有相同逻辑

#### TypeUtil 说明

代码使用的是 Hutool 工具库的 `cn.hutool.core.util.TypeUtil.getTypeArgument()` 方法。

该方法通过 Java 反射的 `ParameterizedType` 机制，解析类声明时指定的泛型参数。

#### 子类必须显式指定泛型的原因

构造函数中直接调用 `getMessageClass()`：

**Stream 监听器：** `AbstractRedisStreamMessageListener.java:51-54`

```java
@SneakyThrows
protected AbstractRedisStreamMessageListener() {
    this.messageType = getMessageClass();
    this.streamKey = messageType.getDeclaredConstructor().newInstance().getStreamKey();
}
```

**Pub/Sub 监听器：** `AbstractRedisChannelMessageListener.java:40-43`

```java
@SneakyThrows
protected AbstractRedisChannelMessageListener() {
    this.messageType = getMessageClass();
    this.channel = messageType.getDeclaredConstructor().newInstance().getChannel();
}
```

#### 结论

| 场景 | 结果 |
|------|------|
| 子类显式指定泛型（如 `MyListener extends AbstractRedisStreamMessageListener<MyMessage>`） | ✅ 正常解析，`getTypeArgument()` 返回 `MyMessage.class` |
| 子类不指定泛型（如 `MyListener extends AbstractRedisStreamMessageListener`） | ❌ 抛出 `IllegalStateException: 类型(xxx) 需要设置消息类型` |
| 类注释说明 | 源码注释明确说明：`消息类型。一定要填写噢，不然会报错` |

#### 泛型解析原理详解

```java
// 正确写法 - 编译时保留泛型信息
public class OrderCreateStreamListener 
    extends AbstractRedisStreamMessageListener<OrderCreateMessage> {
    // TypeUtil 能解析出 OrderCreateMessage.class
}

// 错误写法 - 泛型被擦除
public class OrderCreateStreamListener 
    extends AbstractRedisStreamMessageListener {  // 无泛型
    // TypeUtil.getTypeArgument() 返回 null
    // 抛出 IllegalStateException
}
```

### 2.4 Redis Pub/Sub send 与 Redis Stream send 的可靠性语义差异

#### 发送端实现对比

**代码位置：** `RedisMQTemplate.java:38-64`

```java
// Pub/Sub 发送
public <T extends AbstractRedisChannelMessage> void send(T message) {
    try {
        sendMessageBefore(message);
        redisTemplate.convertAndSend(message.getChannel(), JsonUtils.toJsonString(message));
    } finally {
        sendMessageAfter(message);
    }
}

// Stream 发送
public <T extends AbstractRedisStreamMessage> RecordId send(T message) {
    try {
        sendMessageBefore(message);
        return redisTemplate.opsForStream().add(StreamRecords.newRecord()
                .ofObject(JsonUtils.toJsonString(message))
                .withStreamKey(message.getStreamKey()));
    } finally {
        sendMessageAfter(message);
    }
}
```

#### 可靠性语义对比表

| 维度 | Redis Pub/Sub (PUBLISH) | Redis Stream (XADD) |
|------|------------------------|---------------------|
| **消息持久化** | ❌ 消息发布后立即删除，不保存历史 | ✅ 消息持久化在 Stream 中，可配置保留策略 |
| **消费者确认 (ACK)** | ❌ 无 ACK 机制，发布即忘记 | ✅ 支持 ACK，只有确认后才标记已消费 |
| **消费语义** | **至多一次 (At-most-once)** - 订阅者不在线就丢失 | **至少一次 (At-least-once)** - 未 ACK 的消息保留在 PEL |
| **消费者分组** | ❌ 不支持消费者组，所有订阅者都收到 | ✅ 支持消费者组，实现负载均衡的集群消费 |
| **消息回溯** | ❌ 只能消费订阅之后的消息 | ✅ 可从任意偏移量开始消费 |
| **消息顺序** | ✅ 同一通道内保持发布顺序 | ✅ Stream 严格按插入顺序，支持按时间范围查询 |
| **失败重试** | ❌ 无内置重试机制 | ✅ 通过 Pending List + 定时任务实现重试 |
| **消费者故障恢复** | ❌ 消费者重启后丢失期间消息 | ✅ 未 ACK 的消息在 PEL 中，可被重新认领 |

#### 配置层面的可靠性差异

**代码位置：** `YudaoRedisMQConsumerAutoConfiguration.java:126-129`

```java
StreamMessageListenerContainer.StreamReadRequestBuilder<String> builder = 
    StreamMessageListenerContainer.StreamReadRequest
        .builder(streamOffset).consumer(consumer)
        .autoAcknowledge(false)  // 关键：不自动 ack，业务手动确认
        .cancelOnError(throwable -> false); // 发生异常不取消消费
```

**总结：**
- **Pub/Sub**：适合即时通知、广播场景，不保证消息可靠送达
- **Stream**：适合可靠消息处理、业务解耦，支持 ACK、重发、消费者组

### 2.5 租户或链路追踪拦截器参与时，异常场景下的上下文清理

#### 租户拦截器实现

**代码位置：** `TenantRedisMessageInterceptor.java:18-42`

```java
public class TenantRedisMessageInterceptor implements RedisMessageInterceptor {

    @Override
    public void sendMessageBefore(AbstractRedisMessage message) {
        Long tenantId = TenantContextHolder.getTenantId();
        if (tenantId != null) {
            message.addHeader(HEADER_TENANT_ID, tenantId.toString());
        }
    }

    @Override
    public void consumeMessageBefore(AbstractRedisMessage message) {
        String tenantIdStr = message.getHeader(HEADER_TENANT_ID);
        if (StrUtil.isNotEmpty(tenantIdStr)) {
            TenantContextHolder.setTenantId(Long.valueOf(tenantIdStr));
        }
    }

    @Override
    public void consumeMessageAfter(AbstractRedisMessage message) {
        // 注意，Consumer 是一个逻辑的入口，所以不考虑原本上下文就存在租户编号的情况
        TenantContextHolder.clear();
    }
}
```

#### 异常场景下的执行流程

```
消费异常时的拦截器执行顺序（拦截器列表: [TenantInterceptor, TracerInterceptor]）：

1. consumeMessageBefore(TenantInterceptor)  → 设置租户上下文
2. consumeMessageBefore(TracerInterceptor)  → 开始链路追踪
3. onMessage() 业务逻辑                    → 抛出异常
4. acknowledge()                            → 不执行（因为异常）
5. consumeMessageAfter(TracerInterceptor)   → 倒序，先执行链路追踪清理
6. consumeMessageAfter(TenantInterceptor)   → 再执行租户上下文清理
```

#### 结论

| 场景 | 上下文是否被清理 | 原因 |
|------|-----------------|------|
| **正常消费** | ✅ 清理 | `consumeMessageAfter` 正常执行 |
| **onMessage 抛异常** | ✅ 清理 | `consumeMessageAfter` 在 **finally** 块中执行 |
| **consumeMessageBefore 中抛异常** | ⚠️ **部分未清理** | 如果在设置上下文的拦截器之前抛异常，已设置的上下文不会被清理 |
| **拦截器抛异常（非第一个）** | ✅ **部分清理** | 已执行的 `consumeMessageBefore` 对应的拦截器，其 `consumeMessageAfter` 会倒序执行 |

#### 潜在风险分析

**风险场景：** 拦截器链中间某个拦截器在 `consumeMessageBefore` 中抛出异常

```
假设拦截器顺序: [A, B, C]
执行流程:
  A.consumeMessageBefore()  ✓ 设置上下文 A
  B.consumeMessageBefore()  ✓ 设置上下文 B
  C.consumeMessageBefore()  ✗ 抛异常
  
finally 块执行:
  C.consumeMessageAfter()  → 跳过（C 没成功执行 before）
  B.consumeMessageAfter()  ✓ 清理 B
  A.consumeMessageAfter()  ✓ 清理 A
```

**结论：** 当前实现在异常场景下能够保证**已成功设置的上下文被清理**，因为：
1. `consumeMessageAfter` 在 finally 块中执行
2. 采用倒序执行，对应已执行的 before 逻辑

**注意：** 链路追踪拦截器如果存在（代码库中未找到显式实现），需要遵循相同的模式在 `consumeMessageAfter` 中清理。

### 2.6 消费失败后 pending message 的后续风险分析

#### Pending Message 产生原因

**代码位置：** `AbstractRedisStreamMessageListener.java:63-80`

```java
@Override
public void onMessage(ObjectRecord<String, String> message) {
    try {
        consumeMessageBefore(messageObj);
        this.onMessage(messageObj);  // <-- 这里抛异常
        redisMQTemplate.getRedisTemplate().opsForStream().acknowledge(group, message);
    } finally {
        consumeMessageAfter(messageObj);
    }
}
```

当 `onMessage` 抛异常时：
- **不执行 ack** → 消息保留在 **Pending Entry List (PEL)** 中
- **消费者已认领** → 消息状态为 "已被消费者 X 认领但未确认"

#### RedisPendingMessageResendJob 处理逻辑

**代码位置：** `RedisPendingMessageResendJob.java:43-98`

```java
// 每分钟执行一次
@Scheduled(cron = "35 * * * * ?")
public void messageResend() { ... }

// 核心逻辑：
// 1. 遍历所有监听器的 consumer group
// 2. 检查每个 consumer 的 pending 消息
// 3. 如果消息"空闲时间" > 5 分钟 (EXPIRE_TIME)
// 4. 重新投递到 Stream（创建新的消息）
// 5. 确认原消息（ack）
```

#### 风险分析（不仅是"自动重试"）

**风险 1：消息重复消费**

```
时间线：
T0: 消息 M1 被消费者 A 认领，开始消费
T0.5s: 消费者 A 处理中（耗时较长），未 ack
T1m: 定时任务扫描，发现 M1 空闲 1 分钟 < 5 分钟，跳过
T5.5m: 定时任务扫描，M1 空闲 5.5 分钟 > 5 分钟
       → 重新投递 M1'（新消息 ID），并 ack 原 M1
T6m: 消费者 A 终于处理完成 M1，尝试 ack
       → M1 已被 ack（不会报错，但业务逻辑已执行）
T6.1m: 其他消费者 B 收到 M1'，再次执行业务逻辑

结果：业务逻辑被执行 **2 次**
```

**风险点：**
- 5 分钟超时是硬编码，无法针对不同业务调整
- 没有检查消费者是否存活（可能消费者只是处理慢，而非崩溃）
- 重发是创建**新消息**，原消息被 ack，导致重复

**风险 2：Pending List 内存泄漏**

```
场景：
- 消费者 A 正常运行，但某个消息处理逻辑一直抛异常
- 消息每次消费都失败，不 ack
- 消息一直在 PEL 中

后果：
- PEL 不断累积
- Redis 内存占用持续增长
- 定时任务每次都要扫描这些消息，增加 Redis 压力

更严重的是：
- 如果异常是永久性的（如业务逻辑 bug、数据格式错误）
- 消息会永远留在 PEL 中，直到手动清理
```

**风险 3：没有死信队列（DLQ）机制**

```
对比专业 MQ：
RocketMQ/Kafka/Pulsar 都有：
  - 重试次数限制
  - 达到上限后投递到死信队列
  - 人工介入处理死信

当前实现：
  - 没有重试次数计数
  - 没有死信队列
  - 异常消息会被无限重发（或永远 pending）
```

**风险 4：重发顺序打乱**

```
原始消息顺序：M1, M2, M3

场景：
- M1 消费失败 → pending
- M2 消费成功
- M3 消费失败 → pending

5 分钟后重发：
- M1' 被投递（新消息 ID）
- M3' 被投递

最终消费顺序可能变成：M2, M1', M3'

如果业务依赖严格顺序，会导致数据不一致
```

**风险 5：消费者崩溃与消息认领问题**

```
代码使用 XREADGROUP + autoAcknowledge(false)
但没有使用 XCLAIM 机制

问题：
- 消费者 A 崩溃，它认领的消息留在 PEL 中
- 其他消费者无法自动认领这些消息
- 必须等待 5 分钟，由定时任务重新投递
- 但重新投递是"新消息"，而非"认领原消息"

对比专业实现：
- 应该使用 XCLAIM 让存活的消费者认领超时消息
- 而非重新投递新消息
```

**风险 6：监控和告警缺失**

```
当前实现：
- 只有日志打印
- 没有 Prometheus metrics
- 没有告警机制

问题：
- Pending 消息堆积无法及时发现
- 重复消费次数无法统计
- 异常消费原因无法追踪
```

## 三、代码引用汇总

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| Stream 发送 | `RedisMQTemplate.java` | 54-64 |
| Pub/Sub 发送 | `RedisMQTemplate.java` | 38-46 |
| 拦截器顺序 | `RedisMQTemplate.java` | 75-85 |
| Stream 消费和 ack | `AbstractRedisStreamMessageListener.java` | 63-80 |
| 泛型解析 | `AbstractRedisStreamMessageListener.java` | 94-101 |
| Pub/Sub 消费 | `AbstractRedisChannelMessageListener.java` | 55-64 |
| 租户拦截器 | `TenantRedisMessageInterceptor.java` | 18-42 |
| Pending 重发任务 | `RedisPendingMessageResendJob.java` | 43-98 |
| Consumer 配置 | `YudaoRedisMQConsumerAutoConfiguration.java` | 92-135 |
| Producer 配置 | `YudaoRedisMQProducerAutoConfiguration.java` | 22-29 |

## 四、改进建议

### 4.1 Ack 机制改进

1. 将 ack 逻辑移到 try-catch 外或独立的错误处理分支
2. 区分"可重试异常"和"不可重试异常"
3. 不可重试异常直接 ack 并投递死信队列

### 4.2 Pending Message 处理改进

1. 引入重试次数计数（存储在消息 header 中）
2. 超过最大重试次数后投递到死信队列
3. 使用 `XCLAIM` 而非重新投递来处理超时消息
4. 提供可配置的超时时间和重试策略

### 4.3 监控增强

1. 添加 Pending 消息数量的 metrics
2. 添加消费成功率、重试次数的 metrics
3. Pending 超过阈值时触发告警
