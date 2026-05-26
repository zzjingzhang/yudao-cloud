# yudao-cloud 支付通知任务完整链路追踪分析

## 一、概述

支付通知系统是支付模块的核心组件之一，负责在支付订单、退款、转账等交易成功后，主动回调通知业务方。该系统采用了**异步回调 + 定时重试**的机制，确保通知的可靠性和最终一致性。

### 涉及的核心类协作关系

| 类名 | 职责 | 文件路径 |
|------|------|----------|
| `PayNotifyTaskDO` | 通知任务数据模型，存储任务状态、重试次数等 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/dal/dataobject/notify/PayNotifyTaskDO.java` |
| `PayNotifyLogDO` | 通知日志数据模型，记录每次通知的结果 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/dal/dataobject/notify/PayNotifyLogDO.java` |
| `PayNotifyTypeEnum` | 通知类型枚举（订单/退款/转账） | `yudao-module-pay-api/src/main/java/cn/iocoder/yudao/module/pay/enums/notify/PayNotifyTypeEnum.java` |
| `PayNotifyStatusEnum` | 通知状态枚举（等待/成功/失败/请求成功/请求失败） | `yudao-module-pay-api/src/main/java/cn/iocoder/yudao/module/pay/enums/notify/PayNotifyStatusEnum.java` |
| `PayNotifyTaskMapper` | 通知任务数据库访问层 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/dal/mysql/notify/PayNotifyTaskMapper.java` |
| `PayNotifyLogMapper` | 通知日志数据库访问层 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/dal/mysql/notify/PayNotifyLogMapper.java` |
| `PayNotifyLockRedisDAO` | 分布式锁 Redis 访问层 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/dal/redis/notify/PayNotifyLockRedisDAO.java` |
| `PayNotifyServiceImpl` | 通知服务核心实现 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/service/notify/PayNotifyServiceImpl.java` |
| `PayNotifyJob` | 定时任务，扫描待通知任务 | `yudao-module-pay-server/src/main/java/cn/iocoder/yudao/module/pay/job/notify/PayNotifyJob.java` |

---

## 二、通知任务的创建时机（问题 1）

### 2.1 支付订单成功后创建通知任务

**调用链路**：

```
支付渠道回调 → PayOrderServiceImpl.notifyOrderSuccess() → PayNotifyServiceImpl.createPayNotifyTask()
```

**代码位置**：`PayOrderServiceImpl.java:302-303`

```java
private void notifyOrderSuccess(PayChannelDO channel, PayOrderRespDTO notify) {
    // 1. 更新 PayOrderExtensionDO 支付成功
    PayOrderExtensionDO orderExtension = updateOrderSuccess(notify);
    // 2. 更新 PayOrderDO 支付成功
    Boolean paid = updateOrderSuccess(channel, orderExtension, notify);
    if (paid) { // 如果之前已经成功回调，则直接返回，不用重复记录支付通知记录
        return;
    }
    // 3. 插入支付通知记录
    notifyService.createPayNotifyTask(PayNotifyTypeEnum.ORDER.getType(),
            orderExtension.getOrderId());
}
```

**关键要点**：
- 仅在支付订单**首次从 WAITING 变为 SUCCESS** 时才创建通知任务
- `updateOrderSuccess` 方法使用 `updateByIdAndStatus` 做状态机校验，确保幂等
- 如果支付平台重复回调，`paid` 会为 true，直接返回，避免重复创建通知任务

### 2.2 退款成功/失败后创建通知任务

**调用链路**：

```
支付渠道退款回调 → PayRefundServiceImpl.notifyRefundSuccess()/notifyRefundFailure() 
    → PayNotifyServiceImpl.createPayNotifyTask()
```

**代码位置**：
- `PayRefundServiceImpl.java:245-246`（成功）
- `PayRefundServiceImpl.java:276-277`（失败）

**通知创建场景**：
| 场景 | 方法 | 状态转换 |
|------|------|----------|
| 退款成功 | `notifyRefundSuccess` | WAITING → SUCCESS |
| 退款失败 | `notifyRefundFailure` | WAITING → FAILURE |

**关键要点**：
- 退款成功和失败**都会**创建通知任务
- 退款失败也通知业务方，方便业务方做补偿处理

### 2.3 转账成功/关闭后创建通知任务

**调用链路**：

```
支付渠道转账回调 → PayTransferServiceImpl.notifyTransferSuccess()/notifyTransferClosed()
    → PayNotifyServiceImpl.createPayNotifyTask()
```

**代码位置**：
- `PayTransferServiceImpl.java:210`（成功）
- `PayTransferServiceImpl.java:240`（关闭/失败）

**通知创建场景**：
| 场景 | 方法 | 状态转换 |
|------|------|----------|
| 转账成功 | `notifyTransferSuccess` | WAITING/PROCESSING → SUCCESS |
| 转账关闭 | `notifyTransferClosed` | WAITING/PROCESSING → CLOSED |

### 2.4 通知任务创建的核心逻辑

**代码位置**：`PayNotifyServiceImpl.java:97-129`

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void createPayNotifyTask(Integer type, Long dataId) {
    PayNotifyTaskDO task = new PayNotifyTaskDO().setType(type).setDataId(dataId);
    task.setStatus(PayNotifyStatusEnum.WAITING.getStatus())
        .setNextNotifyTime(LocalDateTime.now())
        .setNotifyTimes(0)
        .setMaxNotifyTimes(PayNotifyTaskDO.NOTIFY_FREQUENCY.length + 1);
    
    // 补充 appId + notifyUrl + merchant* 字段（根据 type 查询不同业务数据）
    if (Objects.equals(task.getType(), PayNotifyTypeEnum.ORDER.getType())) {
        PayOrderDO order = orderService.getOrder(task.getDataId());
        task.setAppId(order.getAppId()).setNotifyUrl(order.getNotifyUrl())
            .setMerchantOrderId(order.getMerchantOrderId());
    } else if (...) { /* 退款/转账类似 */ }
    
    // 执行插入
    notifyTaskMapper.insert(task);
    
    // 必须在事务提交后，再发起任务，否则 PayNotifyTaskDO 还没入库，就提前回调接入的业务
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            // 异步的原因：避免阻塞当前事务，无需等待结果
            getSelf().executeNotifyAsync(task);
        }
    });
}
```

**任务初始化字段**：
| 字段 | 初始值 | 说明 |
|------|--------|------|
| `status` | `WAITING(0)` | 等待通知 |
| `nextNotifyTime` | `LocalDateTime.now()` | 立即可以通知 |
| `notifyTimes` | `0` | 尚未通知过 |
| `maxNotifyTimes` | `9` | 包含首次，共 1 + 8 = 9 次重试机会 |

**关键设计**：
1. **事务提交后回调**：使用 `TransactionSynchronizationManager.afterCommit()` 确保任务入库后才开始回调
2. **异步执行**：使用 `@Async` 异步发起，不阻塞当前支付交易的事务提交

---

## 三、任务状态与字段变化（问题 2）

### 3.1 通知类型枚举

**代码位置**：`PayNotifyTypeEnum.java`

```java
public enum PayNotifyTypeEnum {
    ORDER(1, "支付单"),
    REFUND(2, "退款单"),
    TRANSFER(3, "转账单");
}
```

### 3.2 通知状态枚举

**代码位置**：`PayNotifyStatusEnum.java`

```java
public enum PayNotifyStatusEnum {
    WAITING(0, "等待通知"),           // 初始状态 / 可重试状态
    SUCCESS(10, "通知成功"),          // 终态-成功
    FAILURE(20, "通知失败"),          // 终态-彻底失败（超过最大次数）
    REQUEST_SUCCESS(21, "请求成功，但是结果失败"), // 可重试状态
    REQUEST_FAILURE(22, "请求失败");  // 可重试状态
}
```

**状态说明**：
- **可重试状态**：`WAITING`、`REQUEST_SUCCESS`、`REQUEST_FAILURE`
- **终态**：`SUCCESS`（不再重试）、`FAILURE`（不再重试）
- `REQUEST_SUCCESS` vs `REQUEST_FAILURE`：
  - `REQUEST_SUCCESS`：HTTP 请求成功（200 OK），但返回的 CommonResult.code != 0
  - `REQUEST_FAILURE`：HTTP 请求失败（连接失败、超时、异常等）

### 3.3 重试策略与通知频率

**代码位置**：`PayNotifyTaskDO.java:33-36`

```java
public static final Integer[] NOTIFY_FREQUENCY = new Integer[]{
    15, 15, 30, 180,
    1800, 1800, 1800, 3600
};
```

**重试时间轴**（共 9 次机会）：

| 第几次 | 距上次间隔 | 累计时间 | 说明 |
|--------|-----------|----------|------|
| 第 1 次 | - | 0s（立即） | 创建任务后立即尝试 |
| 第 2 次 | 15s | +15s | 首次失败后 15s |
| 第 3 次 | 15s | +30s | |
| 第 4 次 | 30s | +1min | |
| 第 5 次 | 3min | +4min | |
| 第 6 次 | 30min | +34min | |
| 第 7 次 | 30min | +1h4m | |
| 第 8 次 | 30min | +1h34m | |
| 第 9 次 | 60min | +2h34m | 最后一次尝试 |

### 3.4 字段变化完整生命周期

#### 场景一：通知一次成功

```
创建任务：
  status = WAITING(0)
  notifyTimes = 0
  nextNotifyTime = now()
  lastExecuteTime = null

第 1 次执行：
  执行 HTTP 回调 → 返回 success
  status = SUCCESS(10)
  notifyTimes = 1
  lastExecuteTime = now()
  nextNotifyTime = 保持不变（不再需要）
```

#### 场景二：通知多次后成功

```
创建任务：
  status = WAITING(0)
  notifyTimes = 0
  nextNotifyTime = now()
  lastExecuteTime = null

第 1 次执行（失败）：
  HTTP 请求失败/业务返回失败
  status = REQUEST_SUCCESS(21) 或 REQUEST_FAILURE(22)
  notifyTimes = 1
  lastExecuteTime = now()
  nextNotifyTime = now() + 15s

第 2 次执行（失败）：
  status = REQUEST_SUCCESS(21) 或 REQUEST_FAILURE(22)
  notifyTimes = 2
  lastExecuteTime = now()
  nextNotifyTime = now() + 15s

第 3 次执行（成功）：
  status = SUCCESS(10)
  notifyTimes = 3
  lastExecuteTime = now()
  nextNotifyTime = 保持不变
```

#### 场景三：通知彻底失败（超过最大次数）

```
第 8 次执行（失败）：
  notifyTimes = 8
  nextNotifyTime = now() + 3600s

第 9 次执行（最后机会，仍失败）：
  status = FAILURE(20)
  notifyTimes = 9
  lastExecuteTime = now()
  nextNotifyTime = 保持不变（终态）
```

### 3.5 状态处理核心逻辑

**代码位置**：`PayNotifyServiceImpl.java:269-297`

```java
Integer processNotifyResult(PayNotifyTaskDO task, CommonResult<?> invokeResult, Throwable invokeException) {
    // 每次执行都更新：lastExecuteTime 和 notifyTimes + 1
    PayNotifyTaskDO updateTask = new PayNotifyTaskDO()
            .setId(task.getId())
            .setLastExecuteTime(LocalDateTime.now())
            .setNotifyTimes(task.getNotifyTimes() + 1);

    // 情况一：调用成功
    if (invokeResult != null && invokeResult.isSuccess()) {
        updateTask.setStatus(PayNotifyStatusEnum.SUCCESS.getStatus());
        notifyTaskMapper.updateById(updateTask);
        return updateTask.getStatus();
    }

    // 情况二：调用失败、调用异常
    // 2.1 超过最大回调次数
    if (updateTask.getNotifyTimes() >= PayNotifyTaskDO.NOTIFY_FREQUENCY.length) {
        updateTask.setStatus(PayNotifyStatusEnum.FAILURE.getStatus());
        notifyTaskMapper.updateById(updateTask);
        return updateTask.getStatus();
    }
    // 2.2 未超过最大回调次数
    updateTask.setNextNotifyTime(addTime(Duration.ofSeconds(
        PayNotifyTaskDO.NOTIFY_FREQUENCY[updateTask.getNotifyTimes()])));
    updateTask.setStatus(invokeException != null 
        ? PayNotifyStatusEnum.REQUEST_FAILURE.getStatus()
        : PayNotifyStatusEnum.REQUEST_SUCCESS.getStatus());
    notifyTaskMapper.updateById(updateTask);
    return updateTask.getStatus();
}
```

**状态更新规则**：
| 回调结果 | 异常 | notifyTimes | 新状态 | nextNotifyTime |
|----------|------|-------------|--------|----------------|
| success | - | any | SUCCESS | 不变 |
| 业务失败 | 否 | < 8 | REQUEST_SUCCESS | 按 NOTIFY_FREQUENCY 递增 |
| 业务失败 | 否 | >= 8 | FAILURE | 不变 |
| HTTP 异常 | 是 | < 8 | REQUEST_FAILURE | 按 NOTIFY_FREQUENCY 递增 |
| HTTP 异常 | 是 | >= 8 | FAILURE | 不变 |

---

## 四、分布式锁机制（问题 3）

### 4.1 PayNotifyLockRedisDAO 的作用

**代码位置**：`PayNotifyLockRedisDAO.java:17-38`

```java
@Repository
public class PayNotifyLockRedisDAO {

    @Resource
    private RedissonClient redissonClient;

    public void lock(Long id, Long timeoutMillis, Runnable runnable) {
        String lockKey = formatKey(id);
        RLock lock = redissonClient.getLock(lockKey);
        try {
            lock.lock(timeoutMillis, TimeUnit.MILLISECONDS);
            runnable.run();
        } finally {
            lock.unlock();
        }
    }

    private static String formatKey(Long id) {
        return String.format(PAY_NOTIFY_LOCK, id); // "pay_notify:lock:%d"
    }
}
```

### 4.2 锁的粒度：锁住的是任务，不是业务单

**Redis Key 格式**：`RedisKeyConstants.java:17`

```java
String PAY_NOTIFY_LOCK = "pay_notify:lock:%d";  // 例如：pay_notify:lock:1001
```

**结论**：
- **锁住的是 `PayNotifyTaskDO` 的 ID**，即**通知任务级别**的锁
- **不是**锁住业务单（支付订单/退款单/转账单）的 ID
- 同一个业务单可以创建多个通知任务（理论上，实际通过业务层幂等控制避免）

### 4.3 为什么需要分布式锁？

**锁的使用位置**：`PayNotifyServiceImpl.java:187-203`

```java
public void executeNotify(PayNotifyTaskDO task) {
    // 分布式锁，避免并发问题
    notifyLockCoreRedisDAO.lock(task.getId(), NOTIFY_TIMEOUT_MILLIS, () -> {
        // 校验，当前任务是否已经被通知过
        // 虽然已经通过分布式加锁，但是可能同时满足通知的条件，然后都去获得锁。
        // 此时，第一个执行完后，第二个还是能拿到锁，然后会再执行一次。
        // 因此，此处我们通过 notifyTimes 通知次数是否匹配来判断
        PayNotifyTaskDO dbTask = notifyTaskMapper.selectById(task.getId());
        if (ObjectUtil.notEqual(task.getNotifyTimes(), dbTask.getNotifyTimes())) {
            log.warn("[executeNotifySync][task({}) 任务被忽略，原因是它的通知不是第 ({}) 次]",
                    JsonUtils.toJsonString(task), dbTask.getNotifyTimes());
            return;
        }
        // 执行通知
        getSelf().executeNotify0(dbTask);
    });
}
```

**需要锁的原因**：

| 问题场景 | 描述 | 锁的作用 |
|----------|------|----------|
| 多实例并发 | 多个服务实例同时运行 `PayNotifyJob`，可能查到同一个任务 | 确保同一时刻只有一个实例执行该任务 |
| 同一个任务被多线程处理 | `executeNotify()` 使用线程池，多个线程可能拿到同一任务 | 防止同任务被多次并发回调 |
| 任务重入 | 第一个执行完后，第二个获取锁可能再次执行 | 配合 `notifyTimes` 校验，防止重入 |

**锁 + 数据库校验的双重保障**：
1. **第一层：Redis 分布式锁** → 防止并发获取
2. **第二层：notifyTimes 校验** → 第一个执行后 notifyTimes 已更新，第二个拿到锁后发现不一致，直接跳过

---

## 五、HTTP 回调的各种情况处理（问题 4）

### 5.1 HTTP 请求发起

**代码位置**：`PayNotifyServiceImpl.java:232-259`

```java
private CommonResult<?> executeNotifyInvoke(PayNotifyTaskDO task) {
    // 拼接 body 参数
    Object request;
    if (Objects.equals(task.getType(), PayNotifyTypeEnum.ORDER.getType())) {
        request = PayOrderNotifyReqDTO.builder()
            .merchantOrderId(task.getMerchantOrderId())
            .payOrderId(task.getDataId()).build();
    } else if (Objects.equals(task.getType(), PayNotifyTypeEnum.REFUND.getType())) {
        request = PayRefundNotifyReqDTO.builder()
            .merchantOrderId(task.getMerchantOrderId())
            .merchantRefundId(task.getMerchantRefundId())
            .payRefundId(task.getDataId()).build();
    } else if (Objects.equals(task.getType(), PayNotifyTypeEnum.TRANSFER.getType())) {
        request = PayTransferNotifyReqDTO.builder()
            .merchantTransferId(task.getMerchantTransferId())
            .payTransferId(task.getDataId()).build();
    } else {
        throw new RuntimeException("未知的通知任务类型：" + JsonUtils.toJsonString(task));
    }
    
    // 拼接 header 参数（多租户支持）
    Map<String, String> headers = new HashMap<>();
    TenantUtils.addTenantHeader(headers, task.getTenantId());

    // 发起请求，超时时间 120 秒
    try (HttpResponse response = HttpUtil.createPost(task.getNotifyUrl())
            .body(JsonUtils.toJsonString(request)).addHeaders(headers)
            .timeout((int) NOTIFY_TIMEOUT_MILLIS).execute()) {
        return JsonUtils.parseObject(response.body(), CommonResult.class);
    }
}
```

**请求配置**：
- **超时时间**：`NOTIFY_TIMEOUT = 120` 秒
- **HTTP 方法**：POST
- **Content-Type**：JSON
- **Header**：包含 `tenant-id`（多租户场景）

### 5.2 核心执行流程

**代码位置**：`PayNotifyServiceImpl.java:205-224`

```java
@Transactional(rollbackFor = Exception.class)
public void executeNotify0(PayNotifyTaskDO task) {
    // 发起回调
    CommonResult<?> invokeResult = null;
    Throwable invokeException = null;
    try {
        invokeResult = executeNotifyInvoke(task);
    } catch (Throwable e) {
        invokeException = e;
    }

    // 处理结果（更新 task 状态）
    Integer newStatus = processNotifyResult(task, invokeResult, invokeException);

    // 记录 PayNotifyLog 日志
    String response = invokeException != null 
        ? ExceptionUtil.getRootCauseMessage(invokeException)
        : JsonUtils.toJsonString(invokeResult);
    notifyLogMapper.insert(PayNotifyLogDO.builder()
        .taskId(task.getId())
        .notifyTimes(task.getNotifyTimes() + 1)  // 注意：这里是 +1 后的次数
        .status(newStatus)
        .response(response)
        .build());
}
```

### 5.3 各种情况的详细处理

#### 情况 1：HTTP 回调成功（业务返回 success）

**触发条件**：
- HTTP 状态码 200
- 返回 `CommonResult.success(...)` 即 `code == 0`

**处理逻辑**：

| 操作 | 说明 |
|------|------|
| `status` | 更新为 `SUCCESS(10)` |
| `notifyTimes` | `+1` |
| `lastExecuteTime` | `now()` |
| `nextNotifyTime` | **保持不变**（终态，不再重试） |
| PayNotifyLog | 记录 `status = SUCCESS(10)`，`response = JSON` |

**代码判断**：
```java
if (invokeResult != null && invokeResult.isSuccess()) {
    updateTask.setStatus(PayNotifyStatusEnum.SUCCESS.getStatus());
    notifyTaskMapper.updateById(updateTask);
    return updateTask.getStatus();
}
```

#### 情况 2：HTTP 请求成功，但业务返回失败

**触发条件**：
- HTTP 状态码 200（连接成功、请求成功）
- 返回 `CommonResult.error(...)` 即 `code != 0`
- `invokeException == null`

**处理逻辑**（按 notifyTimes 是否 >= 8 分两种）：

**子情况 2.1：未超过最大次数（notifyTimes < 8）**

| 操作 | 说明 |
|------|------|
| `status` | 更新为 `REQUEST_SUCCESS(21)` |
| `notifyTimes` | `+1` |
| `lastExecuteTime` | `now()` |
| `nextNotifyTime` | `now() + NOTIFY_FREQUENCY[notifyTimes]`（按策略递增） |
| PayNotifyLog | 记录 `status = REQUEST_SUCCESS(21)`，`response = JSON` |

**子情况 2.2：超过最大次数（notifyTimes >= 8）**

| 操作 | 说明 |
|------|------|
| `status` | 更新为 `FAILURE(20)`（彻底失败） |
| `notifyTimes` | `+1` → 最终为 9 |
| `lastExecuteTime` | `now()` |
| `nextNotifyTime` | **保持不变**（终态） |
| PayNotifyLog | 记录 `status = FAILURE(20)`，`response = JSON` |

#### 情况 3：HTTP 请求异常（网络失败、超时、连接拒绝等）

**触发条件**：
- `executeNotifyInvoke()` 抛出异常
- `invokeException != null`

**可能的异常场景**：
| 异常类型 | 举例 |
|----------|------|
| 连接超时 | `ConnectTimeoutException` |
| 读取超时 | `SocketTimeoutException`（120s 未返回） |
| 连接拒绝 | `ConnectException`（服务未启动） |
| 未知主机 | `UnknownHostException` |
| JSON 解析异常 | `JSONException`（返回格式不对） |
| 未知类型 | 代码中的 `throw new RuntimeException("未知的通知任务类型")` |

**处理逻辑**：

**子情况 3.1：未超过最大次数（notifyTimes < 8）**

| 操作 | 说明 |
|------|------|
| `status` | 更新为 `REQUEST_FAILURE(22)` |
| `notifyTimes` | `+1` |
| `lastExecuteTime` | `now()` |
| `nextNotifyTime` | `now() + NOTIFY_FREQUENCY[notifyTimes]` |
| PayNotifyLog | 记录 `status = REQUEST_FAILURE(22)`，`response = 异常堆栈` |

**子情况 3.2：超过最大次数（notifyTimes >= 8）**

| 操作 | 说明 |
|------|------|
| `status` | 更新为 `FAILURE(20)`（彻底失败） |
| `notifyTimes` | `+1` → 最终为 9 |
| `lastExecuteTime` | `now()` |
| `nextNotifyTime` | **保持不变** |
| PayNotifyLog | 记录 `status = FAILURE(20)`，`response = 异常堆栈` |

### 5.4 日志记录（PayNotifyLogDO）

**代码位置**：`PayNotifyLogDO.java:22-51`

```java
public class PayNotifyLogDO extends BaseDO {
    private Long id;
    private Long taskId;           // 关联 PayNotifyTaskDO
    private Integer notifyTimes;   // 第几次通知（注意：是 +1 后的次数）
    private String response;       // HTTP 响应结果或异常消息
    private Integer status;        // 这次通知的结果状态
}
```

**日志记录策略**：
- **每次通知都记录一条日志**，无论成功还是失败
- 同一个 `taskId` 对应多条 `PayNotifyLogDO`（每个 notifyTimes 一条）
- `response` 字段：
  - 正常响应：JSON 字符串（`CommonResult`）
  - 异常：`ExceptionUtil.getRootCauseMessage(invokeException)`（根因消息，非完整堆栈）

**日志查询**：
- `PayNotifyLogMapper.selectListByTaskId(Long taskId)` → 查看某个任务的完整通知历史

---

## 六、幂等保障机制与缺口分析（问题 5）

### 6.1 定时任务调度

**代码位置**：`PayNotifyJob.java:17-31`

```java
@Component
@Slf4j
public class PayNotifyJob {

    @Resource
    private PayNotifyService payNotifyService;

    @XxlJob("payNotifyJob")
    @TenantJob // 多租户
    public String execute() throws Exception {
        int notifyCount = payNotifyService.executeNotify();
        log.info("[execute][执行支付通知 ({}) 个]", notifyCount);
        return StrUtil.format("执行支付通知 ({}) 个",notifyCount);
    }
}
```

**调度特性**：
- 使用 **XXL-Job** 分布式调度
- `@TenantJob`：多租户场景下每个租户独立执行
- 调度频率：由 XXL-Job 配置（通常是 1 分钟或更短）

### 6.2 任务查询与执行

**代码位置**：`PayNotifyTaskMapper.java:26-31`

```java
default List<PayNotifyTaskDO> selectListByNotify() {
    return selectList(new LambdaQueryWrapper<PayNotifyTaskDO>()
            .in(PayNotifyTaskDO::getStatus, 
                PayNotifyStatusEnum.WAITING.getStatus(),
                PayNotifyStatusEnum.REQUEST_SUCCESS.getStatus(), 
                PayNotifyStatusEnum.REQUEST_FAILURE.getStatus())
            .le(PayNotifyTaskDO::getNextNotifyTime, LocalDateTime.now()));
}
```

**查询条件**：
1. **状态条件**：`status in (WAITING, REQUEST_SUCCESS, REQUEST_FAILURE)` → 排除终态
2. **时间条件**：`nextNotifyTime <= now()` → 只查询到达执行时间的任务

**执行流程**：`PayNotifyServiceImpl.java:132-152`

```java
@Override
public int executeNotify() throws InterruptedException {
    // 1. 查询需要通知的任务
    List<PayNotifyTaskDO> tasks = notifyTaskMapper.selectListByNotify();
    if (CollUtil.isEmpty(tasks)) {
        return 0;
    }

    // 2. 遍历，逐个异步通知（线程池）
    CountDownLatch latch = new CountDownLatch(tasks.size());
    tasks.forEach(task -> threadPoolTaskExecutor.execute(() -> {
        try {
            executeNotify(task);  // 加锁执行
        } finally {
            latch.countDown();
        }
    }));
    
    // 3. 等待完成（最多 120 秒）
    awaitExecuteNotify(latch);
    return tasks.size();
}
```

**线程池配置**：`PayJobConfiguration.java:14-26`

```java
@Bean(NOTIFY_THREAD_POOL_TASK_EXECUTOR)
public ThreadPoolTaskExecutor notifyThreadPoolTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(8);
    executor.setMaxPoolSize(16);
    executor.setKeepAliveSeconds(60);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("notify-task-");
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
}
```

### 6.3 重复 Job 执行的幂等保障

**场景**：XXL-Job 可能因网络问题导致同一个任务被重复触发

**保障机制**：

| 层级 | 机制 | 效果 |
|------|------|------|
| 第 1 层 | Redis 分布式锁 `pay_notify:lock:{taskId}` | 同一任务同一时间只能一个线程/实例执行 |
| 第 2 层 | `notifyTimes` 版本号校验 | 第一个执行后 notifyTimes +1，第二个拿到锁后发现不一致，跳过 |
| 第 3 层 | 任务执行后状态变为 SUCCESS/FAILURE 或 nextNotifyTime 延后 | 下次查询时不会再被选中 |

**关键校验代码**：`PayNotifyServiceImpl.java:191-198`

```java
PayNotifyTaskDO dbTask = notifyTaskMapper.selectById(task.getId());
if (ObjectUtil.notEqual(task.getNotifyTimes(), dbTask.getNotifyTimes())) {
    log.warn("[executeNotifySync][task({}) 任务被忽略，原因是它的通知不是第 ({}) 次]",
            JsonUtils.toJsonString(task), dbTask.getNotifyTimes());
    return;
}
```

**工作原理**：
1. 任务查询时拿到 `notifyTimes = N`
2. 执行时校验数据库中的 `notifyTimes` 是否仍为 `N`
3. 如果已被别的线程执行过（变为 `N+1`），则当前线程跳过

### 6.4 并发执行的幂等保障

**场景**：多个服务实例同时运行，或同一实例多个线程同时处理

**保障机制**：

#### 场景 A：同一任务被多个实例/线程查询到

```
实例1: 查询 → 拿到任务 T (notifyTimes=0)
实例2: 查询 → 也拿到任务 T (notifyTimes=0)

实例1: 尝试获取锁 pay_notify:lock:T → 成功
实例2: 尝试获取锁 pay_notify:lock:T → 阻塞

实例1: 执行 → notifyTimes 变为 1 → 释放锁
实例2: 获取锁成功 → 校验 notifyTimes (0 vs 1) → 不一致 → 跳过
```

#### 场景 B：线程池并发处理不同任务

- 线程池使用独立的锁 key（`pay_notify:lock:{taskId}`）
- 不同任务之间互不干扰，可真正并发
- 同一任务即使被多个线程提交，也只会执行一次

### 6.5 服务重启后的幂等保障

**场景**：服务在执行通知过程中宕机

**分析**：

| 时间点 | 状态 | 重启后 |
|--------|------|--------|
| 通知前 | `status=WAITING, notifyTimes=0, nextNotifyTime=now` | 任务仍会被查询到，重新执行 ✅ |
| 通知中（HTTP 已发出） | 未知 | 可能存在问题（见下方缺口分析） |
| 通知后（事务提交前） | 内存中已处理，但 DB 未更新 | 任务仍会被查询到，重新执行 ⚠️ |
| 通知后（事务已提交） | DB 已更新（notifyTimes+1, status 变化） | 不会重复执行 ✅ |

**可能的缺口**：

#### 缺口 1：HTTP 回调已发出，但事务未提交前服务重启

**问题场景**：
```
1. 线程获取锁
2. 执行 HTTP 回调（业务方已收到通知）
3. 准备更新 DB（notifyTimes+1, status...）
4. 服务宕机，事务回滚
5. 服务重启
6. 任务仍满足条件（notifyTimes 还是旧值）
7. 任务被再次执行 → HTTP 回调重复发送
```

**影响**：
- **业务方可能收到重复通知**
- 业务方需要自己做幂等处理（基于 `payOrderId` / `merchantOrderId`）

**当前代码的应对**：
- 没有特殊处理，依赖业务方幂等
- 这是分布式系统的经典"至少一次"（at-least-once）语义

#### 缺口 2：任务执行时间过长，超过锁超时时间

**锁超时时间**：`NOTIFY_TIMEOUT_MILLIS = 120 * 1000 = 120 秒`

**问题场景**：
```
1. 线程A获取锁，开始执行
2. 线程A发起 HTTP 回调（业务方处理很慢）
3. 超过 120 秒，Redisson 锁自动释放（或 watchdog 续期）
4. 线程B获取到同一把锁
5. 线程B校验 notifyTimes（仍相同，因为A还没提交）
6. 线程B也开始执行 HTTP 回调 → 并发回调
```

**分析**：
- Redisson 的 `lock.lock(timeout, unit)` 如果不手动续期，超时后会自动释放
- 但 Redisson 默认有 watchdog 机制（如果持有锁的线程还活着，每 30s 自动续期）
- 如果使用的是 `lock.tryLock(timeout, leaseTime, unit)` 且设置了 leaseTime，则不会自动续期

**当前代码**：
```java
lock.lock(timeoutMillis, TimeUnit.MILLISECONDS);
```
这个 `timeoutMillis` 是获取锁的超时时间？还是锁的持有时间？需要看 Redisson 文档。

实际上 `RLock.lock(long leaseTime, TimeUnit unit)` 中的参数是 **leaseTime（锁的持有时间）**，不是获取超时时间。

所以：
- 锁最多持有 120 秒
- 120 秒后自动释放
- 如果 HTTP 回调超过 120 秒（注意 HTTP 超时也是 120 秒），锁会先释放
- 刚好等于 HTTP 超时时间，理论上不会有问题

#### 缺口 3：任务表没有"执行中"状态

**问题**：
- 没有 `EXECUTING(5, "执行中")` 这样的中间状态
- 查询条件无法区分"待执行"和"执行中"
- 依赖锁 + notifyTimes 版本号机制

**影响**：
- 如果 notifyTimes 校验逻辑有 bug，可能导致问题
- 但当前实现的双重校验（锁 + notifyTimes）已经足够

### 6.6 幂等保障总结

| 场景 | 保障机制 | 是否安全 | 缺口 |
|------|----------|----------|------|
| 重复 Job 触发 | 锁 + notifyTimes 校验 | ✅ 安全 | 无 |
| 多实例并发 | 分布式锁 | ✅ 安全 | 无 |
| 多线程并发 | 分布式锁 | ✅ 安全 | 无 |
| 服务重启（执行前） | 任务状态未更新，会重试 | ✅ 预期行为 | 无 |
| 服务重启（HTTP 发出后，DB 提交前） | 无事务保护 | ⚠️ 可能重复回调 | 业务方需幂等 |
| 锁超时 | HTTP 超时 = 锁超时 | ✅ 刚好匹配 | 无 |
| 业务方重复处理 | 无 | ⚠️ 依赖业务方 | 业务方需幂等 |

**设计哲学**：
- 采用"至少一次"（at-least-once）投递语义
- **不保证**业务方只收到一次通知
- **保证**通知任务一定会被处理（直到成功或彻底失败）
- 依赖**业务方做幂等**（这是行业标准做法）

---

## 七、完整链路时序图

### 7.1 通知任务创建与首次回调

```
支付渠道回调
    |
    v
PayOrderServiceImpl.notifyOrderSuccess()
    |
    +--> updateOrderSuccess() [更新订单状态]
    |
    v
PayNotifyServiceImpl.createPayNotifyTask()
    |
    +--> 创建 PayNotifyTaskDO
    |       status = WAITING
    |       notifyTimes = 0
    |       nextNotifyTime = now()
    |       maxNotifyTimes = 9
    |
    +--> TransactionSynchronizationManager.afterCommit()
    |
    v
@Async executeNotifyAsync(task)
    |
    v
executeNotify(task)
    |
    +--> Redis 锁: pay_notify:lock:{taskId}
    |
    +--> 校验 notifyTimes（防止重入）
    |
    v
executeNotify0(task) [事务]
    |
    +--> HTTP POST 业务方 notifyUrl
    |       timeout = 120s
    |
    +--> processNotifyResult()
    |       |
    |       +--> 成功: status = SUCCESS, notifyTimes = 1
    |       +--> 失败: status = REQUEST_*, nextNotifyTime = now() + 15s
    |
    +--> 插入 PayNotifyLogDO
```

### 7.2 定时任务重试

```
XXL-Job 调度 (每分钟一次)
    |
    v
PayNotifyJob.execute()
    |
    v
PayNotifyServiceImpl.executeNotify()
    |
    +--> selectListByNotify()
    |       status IN (WAITING, REQUEST_SUCCESS, REQUEST_FAILURE)
    |       AND nextNotifyTime <= now()
    |
    +--> 线程池异步执行每个任务
    |
    v
executeNotify(task) [同上]
    |
    +--> 最多重试 9 次
    |       间隔: 15s, 15s, 30s, 3min, 30min, 30min, 30min, 60min
    |
    +--> 全部失败后: status = FAILURE(20)
```

---

## 八、关键设计总结

### 8.1 核心设计原则

| 原则 | 实现方式 |
|------|----------|
| **最终一致性** | 多次重试 + 异步回调 |
| **至少一次投递** | 重试机制 + 无中间状态，可能重复 |
| **业务方幂等** | 系统不保证只一次，依赖业务方处理 |
| **分布式并发安全** | Redis 分布式锁 + notifyTimes 版本号校验 |
| **事务一致性** | 任务创建后事务提交才发起回调 |
| **可观测性** | 每次通知都记录 PayNotifyLogDO |

### 8.2 关键类交互图

```
┌─────────────────────────────────────────────────────────────┐
│                    业务方服务                                │
│                    (HTTP 回调接收方)                          │
└───────────────────────▲─────────────────────────────────────┘
                        │ HTTP POST (notifyUrl)
                        │
┌───────────────────────┴─────────────────────────────────────┐
│                    yudao-module-pay                         │
│                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐ │
│  │ PayOrderImpl │     │ PayRefundImpl│     │PayTransferIm │ │
│  │  (创建任务)  │     │  (创建任务)  │     │  (创建任务)  │ │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘ │
│         │                    │                    │         │
│         └────────────────────┼────────────────────┘         │
│                              │                              │
│                              ▼                              │
│                   ┌──────────────────┐                      │
│                   │ PayNotifyService │                      │
│                   │   (Impl)         │                      │
│                   └────────┬─────────┘                      │
│                            │                                │
│              ┌─────────────┼─────────────┐                  │
│              ▼             ▼             ▼                  │
│      ┌────────────┐ ┌────────────┐ ┌────────────┐          │
│      │ TaskMapper │ │ LogMapper  │ │LockRedisDAO│          │
│      │ (MySQL)    │ │ (MySQL)    │ │ (Redis)    │          │
│      └────────────┘ └────────────┘ └────────────┘          │
│                            ▲                                │
│                            │                                │
│                   ┌────────┴─────────┐                      │
│                   │   PayNotifyJob   │                      │
│                   │   (XXL-Job)      │                      │
│                   └──────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 数据库表字段摘要

**pay_notify_task 表核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| type | INT | 1=订单, 2=退款, 3=转账 |
| data_id | BIGINT | 关联业务单 ID |
| status | INT | 0=WAITING, 10=SUCCESS, 20=FAILURE, 21=REQ_SUCC, 22=REQ_FAIL |
| next_notify_time | DATETIME | 下次通知时间 |
| last_execute_time | DATETIME | 上次执行时间 |
| notify_times | INT | 已通知次数 |
| max_notify_times | INT | 最大通知次数（=9） |
| notify_url | VARCHAR | 回调地址 |

**pay_notify_log 表核心字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | BIGINT | 主键 |
| task_id | BIGINT | 关联 task |
| notify_times | INT | 第几次通知 |
| status | INT | 本次通知的结果状态 |
| response | TEXT | 响应内容或异常消息 |

---

## 九、问答部分（按用户问题整理）

### Q1：支付订单、退款、转账成功后如何创建通知任务？

**支付订单**：
- 入口：`PayOrderServiceImpl.notifyOrderSuccess()`
- 时机：支付渠道回调，订单状态从 `WAITING` → `SUCCESS`
- 仅首次成功时创建，重复回调通过 `updateByIdAndStatus` 幂等控制

**退款**：
- 入口：`PayRefundServiceImpl.notifyRefundSuccess()` / `notifyRefundFailure()`
- 时机：退款成功 **或** 退款失败时都会创建
- 业务方可以知道退款结果（包括失败，便于补偿）

**转账**：
- 入口：`PayTransferServiceImpl.notifyTransferSuccess()` / `notifyTransferClosed()`
- 时机：转账成功 **或** 转账关闭时都会创建

**创建流程**：
1. `createPayNotifyTask(type, dataId)`
2. 查询业务数据填充 `appId`、`notifyUrl`、`merchantOrderId` 等
3. 插入 `PayNotifyTaskDO`（status=WAITING, nextNotifyTime=now）
4. 事务提交后通过 `TransactionSynchronization.afterCommit()` 异步发起首次回调

### Q2：notifyTask 的 status、notifyTimes、nextNotifyTime、lastExecuteTime 如何变化？

**初始状态**：
```
status = WAITING(0)
notifyTimes = 0
nextNotifyTime = now()
lastExecuteTime = null
```

**每次执行**：
- `notifyTimes` 总是 `+1`
- `lastExecuteTime` 总是更新为 `now()`

**成功后**：
```
status = SUCCESS(10)
notifyTimes = N + 1
lastExecuteTime = now()
nextNotifyTime = 不变（不再需要）
```

**失败但可重试**（notifyTimes < 8）：
```
status = REQUEST_SUCCESS(21) 或 REQUEST_FAILURE(22)
notifyTimes = N + 1
lastExecuteTime = now()
nextNotifyTime = now() + NOTIFY_FREQUENCY[N] （按策略递增）
```

**失败且不可重试**（notifyTimes >= 8）：
```
status = FAILURE(20)
notifyTimes = 9
lastExecuteTime = now()
nextNotifyTime = 不变（终态）
```

### Q3：为什么需要 PayNotifyLockRedisDAO，它锁住的是任务还是业务单？

**为什么需要**：
1. **多实例并发**：XXL-Job 在多个服务实例上运行，可能同时查询到同一个任务
2. **多线程并发**：`executeNotify()` 内部使用线程池，多个线程可能处理同一任务
3. **任务重入**：第一个线程执行完后，第二个拿到锁可能再次执行

**锁住的是什么**：
- Redis Key: `pay_notify:lock:{taskId}`
- **锁住的是通知任务（PayNotifyTaskDO 的 ID）**
- **不是**锁住业务单（支付订单/退款单/转账单）

**双重保障**：
1. Redis 锁防止并发进入
2. 拿到锁后再次校验 `notifyTimes` 是否匹配，防止重入

### Q4：业务方 HTTP 回调成功、失败、超时、异常时分别如何记录 log 和安排重试？

| 场景 | 触发条件 | PayNotifyLog.status | PayNotifyTask.status | 是否重试 | nextNotifyTime |
|------|----------|---------------------|---------------------|----------|----------------|
| **成功** | HTTP 200 + CommonResult.code=0 | SUCCESS(10) | SUCCESS(10) | ❌ 否 | 不变 |
| **业务失败** | HTTP 200 + CommonResult.code≠0, 次数<9 | REQUEST_SUCCESS(21) | REQUEST_SUCCESS(21) | ✅ 是 | 递增 |
| **业务失败** | HTTP 200 + CommonResult.code≠0, 次数=9 | FAILURE(20) | FAILURE(20) | ❌ 否 | 不变 |
| **请求失败** | 超时/连接异常/解析异常, 次数<9 | REQUEST_FAILURE(22) | REQUEST_FAILURE(22) | ✅ 是 | 递增 |
| **请求失败** | 超时/连接异常/解析异常, 次数=9 | FAILURE(20) | FAILURE(20) | ❌ 否 | 不变 |

**PayNotifyLog 记录**：
- 每次通知都记录一条
- `response` 字段：成功/业务失败时存 JSON，异常时存 `ExceptionUtil.getRootCauseMessage()`

### Q5：重复 Job 执行、并发执行、服务重启后分别有什么幂等保障和缺口？

**重复 Job 执行**：
- ✅ 保障：Redis 锁 + `notifyTimes` 版本号校验
- 第一个执行后 notifyTimes 已更新，第二个拿到锁也会跳过

**并发执行**（多实例/多线程）：
- ✅ 保障：Redis 锁（同一 taskId 同一时间只能一个执行）
- 不同任务的锁互不干扰，可真正并发

**服务重启**：
- ✅ 执行前重启：任务状态未变，会被重新查询执行（预期行为）
- ⚠️ **缺口**：HTTP 已发出但 DB 未提交前重启 → 可能导致业务方收到重复通知
- 设计哲学："至少一次"投递，**依赖业务方做幂等**

**其他潜在缺口**：
- 没有 "执行中" 中间状态，但锁 + notifyTimes 双重校验已足够
- 锁超时 = HTTP 超时（都是 120s），刚好匹配，无额外问题

---

*文档生成时间：2026-05-12*
*基于 yudao-cloud 项目支付模块源码分析*
