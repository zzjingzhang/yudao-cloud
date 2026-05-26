# yudao-cloud 支付订单系统分析

## 一、核心组件职责分析

### 1. PayOrderDO 与 PayOrderExtensionDO 的职责

#### PayOrderDO (支付订单主表)
**核心职责**：代表业务层面的支付订单，记录支付交易的最终状态。

**关键属性**：
- 商户信息：`appId`、`merchantOrderId`（商户订单号，唯一标识业务订单）
- 商品信息：`subject`、`body`
- 订单金额：`price`（总金额）、`refundPrice`（已退款金额）
- 支付状态：`status`（WAITING/SUCCESS/REFUND/CLOSED）
- 成功支付信息：`successTime`、`extensionId`（关联成功的拓展单）、`no`（外部订单号）
- 渠道信息（支付成功后回填）：`channelId`、`channelCode`、`channelOrderNo`

**状态流转**：
```
WAITING(0) → SUCCESS(10) → REFUND(20)
    ↓
  CLOSED(30)
```

#### PayOrderExtensionDO (支付订单拓展表)
**核心职责**：记录每一次与支付渠道的交互调用。

**关键属性**：
- 外部订单号：`no`（调用渠道时使用的 out_trade_no，每次调用生成新的）
- 关联关系：`orderId`（关联 PayOrderDO）、`channelId`、`channelCode`
- 渠道交互状态：`status`（WAITING/SUCCESS/CLOSED）
- 渠道交互信息：`channelExtras`（渠道额外参数）、`channelErrorCode`、`channelErrorMsg`、`channelNotifyData`（渠道回调原始数据）

**设计要点**：
- 每调用一次支付渠道，就插入一条新的拓展记录
- 同一 PayOrderDO 可以有多个 PayOrderExtensionDO
- 只有最终支付成功的那个 extensionId 会回填到 PayOrderDO

---

### 2. PayOrderMapper 与 PayOrderExtensionMapper

#### PayOrderMapper
- `selectByAppIdAndMerchantOrderId(appId, merchantOrderId)`：通过商户订单号查询（防止重复创建）
- `selectByNo(no)`：通过外部订单号查询
- **`updateByIdAndStatus(id, status, update)`**：**乐观锁更新**，仅当当前状态匹配时才执行更新
- `selectListByStatusAndExpireTimeLt(status, expireTime)`：查询过期订单

#### PayOrderExtensionMapper
- `selectByNo(no)`：通过外部订单号查询（渠道回调时使用）
- **`updateByIdAndStatus(id, status, update)`**：**乐观锁更新**，核心防并发手段
- `selectListByOrderId(orderId)`：查询订单的所有拓展记录
- `selectListByStatusAndCreateTimeGe(status, minCreateTime)`：定时同步时查询待支付的拓展单

**关键设计**：两个 Mapper 都提供了 `updateByIdAndStatus` 方法，这是实现乐观锁、防止并发更新的核心机制。

---

## 二、为何一笔订单对应多条 Extension

**原因分析**：

### 场景一：用户重复点击支付
用户在支付页面多次点击"立即支付"按钮，每次点击都会：
1. 调用 `submitOrder` 方法
2. 插入新的 `PayOrderExtensionDO`（生成新的 `no` 作为渠道订单号）
3. 调用不同/相同的支付渠道

这样就会产生多条 extension 记录，但只有其中一条能最终支付成功。

### 场景二：切换支付渠道
用户先选择微信支付，未完成支付；后切换到支付宝支付，重新发起支付：
- 第一次：微信渠道 → Extension 1
- 第二次：支付宝渠道 → Extension 2

### 场景三：渠道调用失败重试
支付渠道调用失败（网络超时、渠道系统异常），业务系统发起重试：
- 第一次尝试 → Extension 1（可能渠道端实际已创建，但本地认为失败）
- 重试调用 → Extension 2

### 设计优势
1. **幂等性保障**：每次渠道调用使用独立的 `no`，避免重复支付
2. **可追溯性**：完整记录每一次渠道交互，便于排查问题
3. **并发安全**：通过乐观锁控制状态更新，只有一条 extension 能成功
4. **支付渠道隔离**：同一订单可尝试多个支付渠道，互不干扰

---

## 三、submitOrder 流程分析

### 核心方法执行顺序

```java
@Transactional  // 注意：实际没有使用事务，避免渠道调用失败回滚 Extension
public PayOrderSubmitRespVO submitOrder(PayOrderSubmitReqVO reqVO, String userIp)
```

**执行顺序**：

#### 阶段一：校验（Validate）
```
1. validateOrderCanSubmit(orderId)
   ├─ 查询 PayOrderDO 是否存在
   ├─ 校验状态：必须是 WAITING
   │  ├─ 如果是 SUCCESS → 抛出 PAY_ORDER_STATUS_IS_SUCCESS
   │  └─ 如果是其他状态 → 抛出 PAY_ORDER_STATUS_IS_NOT_WAITING
   ├─ 校验是否过期：expireTime > now
   └─ validateOrderActuallyPaid(orderId) 【关键！】
      ├─ 查询所有已存在的 extension
      ├─ 情况一：如果某条 extension 状态已是 SUCCESS → 抛异常
      └─ 情况二：调用渠道查询，如果某条 extension 渠道端已支付 → 抛异常

2. validateChannelCanSubmit(appId, channelCode)
   ├─ 校验 App 是否有效
   ├─ 校验渠道是否配置有效
   └─ 获取 PayClient（确保渠道可用）
```

**关键点**：金额校验在哪里？
- 金额是在 `createOrder` 时确定的，`submitOrder` 阶段不再变更订单金额
- 但在 `validateOrderActuallyPaid` 中会查询所有 extension 的支付状态，避免已支付的订单被重复提交

#### 阶段二：创建 Extension 记录
```
3. 生成唯一订单号 no（Redis 自增）
4. 构建 PayOrderExtensionDO
   ├─ status = WAITING
   ├─ channelId, channelCode = 用户选择的渠道
   ├─ no = 新生成的外部订单号
   └─ orderId = 主订单 ID
5. 插入数据库（orderExtensionMapper.insert）
```

**设计**：先插入 extension 再调用渠道，保证即使渠道调用失败，本地也有记录可追溯。

#### 阶段三：调用渠道接口
```
6. 构建 PayOrderUnifiedReqDTO
   ├─ outTradeNo = extension.no 【关键！使用 extension 的 no】
   ├─ subject, body, price = 从 PayOrderDO 获取
   ├─ notifyUrl = 渠道回调地址（包含 channelId）
   └─ expireTime = 订单过期时间
7. client.unifiedOrder(unifiedOrderReqDTO) 调用支付渠道
```

**关键点**：使用 `extension.no` 而非 `order.id` 作为渠道订单号，实现多 extension 的隔离。

#### 阶段四：处理渠道响应
```
8. 若渠道返回支付成功（例如付款码支付）：
   ├─ getSelf().notifyOrder(channel, unifiedOrderResp)
   │  └─ 触发支付成功流程（更新 extension + order 状态）
   └─ try-catch 捕获并发异常（可能与回调同时到达）

9. 若渠道返回错误码：
   └─ 抛出业务异常 PAY_ORDER_SUBMIT_CHANNEL_ERROR

10. 重新查询 PayOrderDO 最新状态
11. 返回提交结果
```

---

## 四、渠道响应状态与本地状态更新

### 状态处理流程图

```
渠道响应
    │
    ├─ 成功（SUCCESS）
    │      │
    │      ▼
    │   notifyOrder()
    │      │
    │      ├─ updateOrderSuccess(notify)  → 更新 Extension
    │      │    ├─ 查询 extension.byNo(outTradeNo)
    │      │    ├─ 若已是 SUCCESS → 直接返回（幂等）
    │      │    ├─ 若不是 WAITING → 抛异常
    │      │    └─ 乐观锁更新: updateByIdAndStatus(WAITING → SUCCESS)
    │      │         └─ 记录 channelNotifyData
    │      │
    │      └─ updateOrderSuccess(channel, extension, notify) → 更新 Order
    │           ├─ 查询 order.byId(extension.orderId)
    │           ├─ 若已是 SUCCESS 且 extensionId 匹配 → 返回 true（重复回调）
    │           ├─ 若不是 WAITING → 抛异常
    │           └─ 乐观锁更新: updateByIdAndStatus(WAITING → SUCCESS)
    │                ├─ 回填 channelId, channelCode, channelOrderNo
    │                ├─ 回填 extensionId, no（成功的那个 extension）
    │                ├─ 计算并回填 channelFeeRate, channelFeePrice
    │                └─ 设置 successTime
    │
    ├─ 等待（WAITING）
    │      │
    │      ▼
    │   无需处理，保持本地状态为 WAITING
    │   等待：① 渠道异步回调 ② PayOrderSyncJob 定时同步
    │
    ├─ 关闭/失败（CLOSED）
    │      │
    │      ▼
    │   notifyOrderClosed()
    │      │
    │      └─ updateOrderExtensionClosed()
    │           ├─ 查询 extension.byNo(outTradeNo)
    │           ├─ 若已是 CLOSED → 直接返回
    │           ├─ 若已是 SUCCESS → 不更新（可能是全部退款导致）
    │           ├─ 若不是 WAITING → 抛异常
    │           └─ 乐观锁更新: updateByIdAndStatus(WAITING → CLOSED)
    │                ├─ 记录 channelNotifyData
    │                └─ 记录 channelErrorCode, channelErrorMsg
    │
    │      【注意】：CLOSED 状态只更新 Extension，不更新 Order
    │      Order 的关闭由 expireOrder() 定时任务处理
    │
    └─ 异常（Exception）
           │
           ▼
        submitOrder 中：
        ├─ 【已插入的 Extension 不会回滚】（方法无 @Transactional）
        ├─ Extension 状态保持 WAITING
        ├─ Order 状态保持 WAITING
        └─ 后续可通过：
             ① 用户重新支付 → 生成新 Extension
             ② PayOrderSyncJob 定时查询该 Extension 的实际状态
```

### 状态变更总结表

| 渠道响应 | PayOrderExtensionDO 变化 | PayOrderDO 变化 |
|---------|------------------------|----------------|
| SUCCESS | WAITING → SUCCESS，记录回调数据 | WAITING → SUCCESS，回填渠道信息、extensionId、successTime |
| WAITING | 无变化（保持 WAITING） | 无变化（保持 WAITING） |
| CLOSED | WAITING → CLOSED，记录错误信息 | 无变化（保持 WAITING，等待过期任务处理） |
| 异常 | 保持 WAITING（已插入不回滚） | 保持 WAITING |

---

## 五、同步 Job 与异步通知的协作与竞态

### 双保险机制

```
┌─────────────────────────────────────────────────────────────┐
│                    支付状态同步双保险                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────┐           ┌────────────────┐           │
│  │  异步通知回调    │           │  PayOrderSyncJob│           │
│  │  (实时性高)      │           │  (定时兜底)      │           │
│  └────────┬───────┘           └────────┬───────┘           │
│           │                            │                   │
│           │ notifyOrder(channelId,     │ syncOrder()        │
│           │   notify)                  │   ↓                │
│           │                            │ 查 WAITING 状态    │
│           │                            │ 的 extension       │
│           │                            │   ↓                │
│           │                            │ 调用渠道查询        │
│           │                            │   ↓                │
│           │                            │ notifyOrder()      │
│           │                            │                   │
│           └──────────────┬─────────────┘                   │
│                          │                                 │
│                          ▼                                 │
│                 同一套更新逻辑                               │
│                 updateByIdAndStatus                        │
│                 (乐观锁保证幂等)                            │
└─────────────────────────────────────────────────────────────┘
```

### 两者如何协作

**1. 触发时机不同**
- **异步通知**：支付渠道在支付状态变更时主动推送（实时）
- **同步 Job**：每 X 分钟执行一次（默认 10 分钟内的订单），兜底保障

**2. 处理范围不同**
- **异步通知**：只处理当前回调对应的那一条 extension
- **同步 Job**：扫描所有近期的 WAITING 状态 extension，逐一查询渠道状态

**3. 竞态场景**

**场景 A：回调先到，Job 后执行**
```
时间线：
T1: 用户支付成功
T2: 渠道异步通知 → Extension: WAITING→SUCCESS, Order: WAITING→SUCCESS
T3: PayOrderSyncJob 执行
    ├─ 扫描时 extension 已非 WAITING，不会被选中
    └─ 无操作，安全
```

**场景 B：Job 先到，回调后到**
```
时间线：
T1: 用户支付成功
T2: PayOrderSyncJob 执行
    ├─ 查询 extension 为 WAITING
    ├─ 调用渠道 getOrder() → 返回 SUCCESS
    ├─ notifyOrder() 执行
    │   ├─ Extension: WAITING→SUCCESS（乐观锁更新成功）
    │   └─ Order: WAITING→SUCCESS（乐观锁更新成功）
    └─ 返回
T3: 渠道异步通知到达
    ├─ notifyOrder() 执行
    ├─ updateOrderSuccess() 查询 extension
    │   └─ extension.status 已是 SUCCESS → 直接返回（幂等）
    └─ 无重复操作，安全
```

**场景 C：两者同时到达（真正的竞态）**
```
时间线：
T1: 用户支付成功
T2: 回调 & Job 同时到达，并发执行 notifyOrder()

线程 A（回调）：
  读取 extension.status = WAITING
  准备更新 WAITING → SUCCESS

线程 B（Job）：
  读取 extension.status = WAITING
  准备更新 WAITING → SUCCESS

乐观锁发挥作用：
  UPDATE pay_order_extension SET status=SUCCESS, ... 
  WHERE id=? AND status=WAITING
  
  只有 1 条 SQL 能执行成功（updateCounts = 1）
  另 1 条 SQL 执行失败（updateCounts = 0）→ 抛异常

  对于 Order 更新同理
```

---

## 六、竞态时序分析与防护

### 构造复杂竞态场景

**场景**：用户重复点击支付 + 渠道慢回调 + 定时同步

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    复杂竞态时序图（时间轴从左到右）                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  时间    操作                          Extension 1  Extension 2  Order  │
│  ───    ───                          ───────────  ───────────  ─────  │
│   T0    createOrder()                ──           ──           WAITING  │
│         (订单创建成功)                                                       │
│                                                                         │
│   T1    【用户点击1】submitOrder()                                         │
│         ├─ validateOrderCanSubmit()                                     │
│         │  └─ 无 extension，通过                                          │
│         ├─ 插入 Extension 1               WAITING     ──           WAITING  │
│         ├─ 调用微信 unifiedOrder()                                     │
│         │  └─ 【微信处理中，响应慢】                                      │
│         └─ 线程 A 等待渠道响应                                           │
│                                                                         │
│   T2    【用户点击2】submitOrder()  ← 重复点击！                           │
│         ├─ validateOrderCanSubmit()                                     │
│         │  ├─ 查到 Extension 1 (WAITING)                                │
│         │  ├─ validateOrderActuallyPaid()                               │
│         │  │  ├─ Extension 1.status=WAITING ✓                           │
│         │  │  └─ 调用微信 getOrder(E1.no)                               │
│         │  │     └─ 微信尚未完成支付 → 返回 WAITING                       │
│         │  └─ 校验通过 ✅                                                │
│         ├─ 插入 Extension 2               WAITING     WAITING      WAITING  │
│         ├─ 调用支付宝 unifiedOrder()                                   │
│         │  └─ 【支付宝快速响应 WAITING】                                 │
│         └─ 返回支付二维码                                                │
│                                                                         │
│   T3    微信支付成功，但【回调延迟】                                       │
│         (微信侧已成功，但未推送通知)                                     │
│                                                                         │
│   T4    PayOrderSyncJob 定时执行                                         │
│         ├─ 扫描 10 分钟内的 WAITING extension                            │
│         │  ├─ Extension 1 (WAITING)                                    │
│         │  └─ Extension 2 (WAITING)                                    │
│         ├─ 处理 Extension 1：                                            │
│         │  ├─ 调用微信 getOrder(E1.no) → 返回 SUCCESS                    │
│         │  └─ notifyOrder() 执行                                        │
│         │     ├─ Extension 1: WAITING→SUCCESS  ✅                      │
│         │     └─ Order: WAITING→SUCCESS, extensionId=E1  ✅           │
│         │         WAITING     SUCCESS      SUCCESS(E1)                  │
│         └─ 处理 Extension 2：                                            │
│            ├─ 调用支付宝 getOrder(E2.no) → 返回 WAITING                  │
│            └─ 跳过，保持 WAITING                                          │
│                                                                         │
│   T5    【微信慢回调到达】                                                 │
│         ├─ notifyOrder(E1.no, SUCCESS)                                  │
│         ├─ updateOrderSuccess(notify)                                   │
│         │  └─ 查询 E1: status=SUCCESS → 直接返回（幂等）                  │
│         └─ 无重复操作，安全 ✅                                           │
│                                                                         │
│   T6    用户在支付宝端完成支付 → 支付宝回调                               │
│         ├─ notifyOrder(E2.no, SUCCESS)                                  │
│         ├─ updateOrderSuccess(notify)                                   │
│         │  └─ 查询 E2: status=WAITING → 尝试更新                         │
│         │     updateByIdAndStatus(E2.id, WAITING, SUCCESS)              │
│         │     ✅ E2 更新成功                                           │
│         │         SUCCESS     SUCCESS      SUCCESS(E1)                  │
│         └─ updateOrderSuccess(channel, E2, notify)                      │
│            ├─ 查询 Order: status=SUCCESS                               │
│            │  且 extensionId=E1 ≠ E2.id                                │
│            └─ 抛异常 PAY_ORDER_STATUS_IS_NOT_WAITING                    │
│               ❌ Order 已被 E1 成功占用                                  │
│               【支付宝支付的钱会原路退回或需人工处理】                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 系统中的防护措施

#### 1. validateOrderCanSubmit 阶段
```java
void validateOrderActuallyPaid(Long id) {
    // 检查一：本地 extension 是否已支付
    if (PayOrderStatusEnum.isSuccess(extension.getStatus())) {
        throw exception(PAY_ORDER_EXTENSION_IS_PAID);
    }
    // 检查二：主动调用渠道查询，防止回调延迟
    PayOrderRespDTO respDTO = payClient.getOrder(extension.getNo());
    if (respDTO != null && PayOrderStatusEnum.isSuccess(respDTO.getStatus())) {
        throw exception(PAY_ORDER_EXTENSION_IS_PAID);
    }
}
```
**作用**：防止 T2 时刻用户重复点击时，Extension 1 已在渠道端支付成功。

#### 2. 乐观锁 updateByIdAndStatus
```java
// Extension 更新
int updateCounts = orderExtensionMapper.updateByIdAndStatus(
    extension.getId(), 
    PayOrderStatusEnum.WAITING.getStatus(),  // 条件：当前必须是 WAITING
    PayOrderExtensionDO.builder().status(SUCCESS).build()
);
if (updateCounts == 0) {
    throw exception(PAY_ORDER_EXTENSION_STATUS_IS_NOT_WAITING);
}

// Order 更新同理
```
**作用**：T4 时刻 Job 和 T5 时刻回调的并发更新只有一个能成功。

#### 3. Order 更新时的 extensionId 校验
```java
if (PayOrderStatusEnum.isSuccess(order.getStatus()) 
        && Objects.equals(order.getExtensionId(), orderExtension.getId())) {
    return true; // 同一个 extension 的重复回调，安全
}
if (!PayOrderStatusEnum.WAITING.getStatus().equals(order.getStatus())) {
    throw exception(PAY_ORDER_STATUS_IS_NOT_WAITING);
}
```
**作用**：T6 时刻检测到 Order 已被其他 extension 成功支付，拒绝重复更新。

### 仍需特别小心的地方

#### 风险点 1：validateOrderActuallyPaid 的查询窗口期
```
T2.1 线程 B 调用 getOrder(E1.no) → 微信返回 WAITING（实际上支付正在处理中）
T2.2 线程 B 校验通过，创建 Extension 2
T2.3 就在此时，微信支付完成
T2.4 但 Extension 2 已经创建了
```
**问题**：T2 时刻的查询是"快照"，查询通过不代表之后不会支付成功。

**缓解**：即使创建了 Extension 2，后续如果 E1 先成功，E2 的回调会被拒绝。但用户可能付了两笔钱（微信 + 支付宝）。

#### 风险点 2：渠道回调顺序
```
实际支付顺序：用户先完成微信支付，后完成支付宝支付
回调到达顺序：支付宝回调先到，微信回调后到
```
**问题**：Extension 2 (支付宝) 先更新 Order 为 SUCCESS，Extension 1 (微信) 的回调会失败。但用户实际通过微信完成了首次支付意图。

#### 风险点 3：多笔支付成功后的资金问题
在 T6 场景中：
- Extension 1 (微信)：支付成功 ✅，Order 关联 ✅
- Extension 2 (支付宝)：支付成功 ✅，Order 拒绝关联 ❌

**结果**：用户实际上付了两笔钱，但系统只记录了一笔成功。

**需要额外处理**：
1. 支付渠道侧的重复支付检测
2. 业务系统需要监听所有 extension 的支付成功事件
3. 发现多付时触发退款流程
4. 人工对账机制

#### 风险点 4：Extension 与 Order 的事务一致性
```java
// notifyOrderSuccess 方法
private void notifyOrderSuccess(PayChannelDO channel, PayOrderRespDTO notify) {
    updateOrderSuccess(notify);        // 更新 Extension（可能成功）
    updateOrderSuccess(channel, ...);  // 更新 Order（可能失败）
    // 如果第一步成功、第二步失败 → 数据不一致！
}
```
**问题**：虽然方法有 `@Transactional`，但要注意：
- 异常会回滚事务 ✓
- 但如果是业务异常（如 Order 已被其他 extension 更新），Extension 的更新也会回滚吗？
- 查看代码：抛异常后 Spring 事务会回滚 ✓

#### 风险点 5：PayOrderSyncJob 的关闭策略
```java
// syncOrder 方法中
if (PayOrderStatusEnum.isClosed(respDTO.getStatus())) {
    return false;  // 渠道返回关闭，本地不做处理！
}
```
**设计理由**：
- 短时间内渠道可能还没创建订单，查询返回"不存在"（被映射为 CLOSED）
- 如果此时本地关闭，后续用户实际支付成功但回调无法处理
- 所以**宁可不关闭，也不误关闭**

**风险**：如果渠道确实关闭了支付，本地 extension 会一直 WAITING，直到 expireOrder 任务处理。

---

## 七、总结

### 核心设计思想

1. **订单与渠道调用分离**：PayOrderDO 记录业务结果，PayOrderExtensionDO 记录每次渠道交互
2. **乐观锁保障幂等**：`updateByIdAndStatus` 是核心防并发手段
3. **双保险同步机制**：异步通知实时性 + 定时 Job 兜底
4. **先落库再调渠道**：extension 不回滚，保证可追溯

### 状态流转核心

```
用户点击支付
    ↓
Extension 插入 (WAITING)
    ↓
渠道调用 → 返回 WAITING/SUCCESS/CLOSED
    ↓
    ├─ SUCCESS：Extension→SUCCESS → Order→SUCCESS（回填 extensionId）
    ├─ WAITING：等待回调 / Job 同步
    └─ CLOSED/异常：Extension 保持 WAITING（不回滚），后续可重试
```

### 竞态防护清单

| 防护点 | 位置 | 作用 |
|-------|------|------|
| validateOrderActuallyPaid | submitOrder 前 | 防止已支付订单重复提交 |
| updateByIdAndStatus (Extension) | 所有状态更新 | Extension 级别乐观锁 |
| updateByIdAndStatus (Order) | 所有状态更新 | Order 级别乐观锁 |
| extensionId 匹配校验 | Order 更新时 | 防止不同 extension 互相覆盖 |
| SUCCESS 状态提前返回 | 所有更新方法 | 幂等性，重复回调直接忽略 |

### 需要关注的风险

1. **用户多渠道支付**：同一订单多个 extension 都支付成功的资金问题
2. **查询窗口期**：validateOrderActuallyPaid 是快照式检查
3. **回调顺序**：先完成的支付不一定先回调
4. **关闭策略保守**：宁可不关也不误关，但可能导致长时间 WAITING

这套设计在技术层面很好地解决了数据一致性问题，但业务层面（如重复支付退款）仍需额外处理。
