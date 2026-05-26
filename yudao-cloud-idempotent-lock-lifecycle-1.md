# yudao-cloud @Idempotent 完整运行语义分析

## 一、协作架构总览

@Idempotent 是 yudao-cloud 中用于防止重复请求的注解，其核心协作流程如下：

```
请求 → IdempotentAspect (AOP 切面)
         ↓
    1. 获取 @Idempotent 注解配置
         ↓
    2. 从 keyResolvers Map 获取对应解析器
         ↓
    3. 调用 IdempotentKeyResolver 解析唯一 key
         ↓
    4. 调用 IdempotentRedisDAO.setIfAbsent() 尝试写入 Redis
         ├─ 成功 → 执行业务方法
         └─ 失败 → 抛出 ServiceException（重复请求）
         ↓
    5. 异常处理：根据 deleteKeyWhenException 决定是否删除 key
```

涉及的核心组件：
- **Idempotent 注解**：配置幂等策略（超时时间、key 解析器、异常时是否删 key 等）
- **IdempotentAspect**：AOP 切面，协调整个幂等流程
- **IdempotentRedisDAO**：封装 Redis 操作（setIfAbsent、delete）
- **IdempotentKeyResolver 体系**：
  - DefaultIdempotentKeyResolver：基于方法名+参数的全局级别
  - UserIdempotentKeyResolver：基于用户ID的用户级别
  - ExpressionIdempotentKeyResolver：基于 SpEL 表达式的自定义级别

---

## 二、问题详细分析

### 1）keyResolvers Map 是如何按 resolver class 匹配的，缺失时抛什么

**匹配机制：**

在 `IdempotentAspect.java:34-36` 的构造函数中：
```java
public IdempotentAspect(List<IdempotentKeyResolver> keyResolvers, IdempotentRedisDAO idempotentRedisDAO) {
    this.keyResolvers = CollectionUtils.convertMap(keyResolvers, IdempotentKeyResolver::getClass);
    this.idempotentRedisDAO = idempotentRedisDAO;
}
```

- Spring 自动注入所有 `IdempotentKeyResolver` 实现类到 `List`
- 通过 `CollectionUtils.convertMap()` 转换为 Map，**Key 是实现类本身（`Class` 对象）**，Value 是实例
- 匹配时使用 `idempotent.keyResolver()` 直接从 Map 中 get

**缺失时的行为：**

在 `IdempotentAspect.java:42-43`：
```java
IdempotentKeyResolver keyResolver = keyResolvers.get(idempotent.keyResolver());
Assert.notNull(keyResolver, "找不到对应的 IdempotentKeyResolver");
```

- 如果 Map 中找不到对应的 resolver class，`keyResolver` 为 null
- `Assert.notNull()` 抛出 **`IllegalArgumentException`**
- 异常消息：**"找不到对应的 IdempotentKeyResolver"**

**注意：** 由于 Map 的 Key 是 `getClass()`（实际类对象），如果注解引用的是接口或未注册的自定义解析器，会匹配失败。

---

### 2）setIfAbsent 成功和失败分别发生什么

**setIfAbsent 实现（IdempotentRedisDAO.java:27-30）：**
```java
public Boolean setIfAbsent(String key, long timeout, TimeUnit timeUnit) {
    String redisKey = formatKey(key);
    return redisTemplate.opsForValue().setIfAbsent(redisKey, "", timeout, timeUnit);
}
```

使用 Redis 的 `SETNX` 语义（SET if Not eXists），并同时设置过期时间。

**成功场景（返回 true）：**
```
请求到达 → 解析 key → Redis 中无此 key → setIfAbsent 成功
                                            ↓
                                      执行业务方法 joinPoint.proceed()
                                            ↓
                                      正常返回 / 抛异常
```

- **成功的含义**：当前请求抢到了"执行权"，是该 key 在超时窗口内的第一个请求
- 后续流程：进入 try 块执行业务逻辑，Redis key 保留（正常返回时不删除）

**失败场景（返回 false）：**
```
请求到达 → 解析 key → Redis 中已有此 key → setIfAbsent 失败
                                            ↓
                                      记录 info 级别日志
                                            ↓
                                      抛出 ServiceException
```

在 `IdempotentAspect.java:50-53`：
```java
if (!success) {
    log.info("[aroundPointCut][方法({}) 参数({}) 存在重复请求]", joinPoint.getSignature().toString(), joinPoint.getArgs());
    throw new ServiceException(GlobalErrorCodeConstants.REPEATED_REQUESTS.getCode(), idempotent.message());
}
```

- **失败的含义**：该 key 已被其他请求占用，当前请求被判定为"重复请求"
- 抛出 **`ServiceException`**，错误码为 `GlobalErrorCodeConstants.REPEATED_REQUESTS`
- 错误消息使用注解中配置的 `message()`（默认："重复请求，请稍后重试"）

---

### 3）业务方法正常返回后 Redis key 是否删除，为什么

**答案：不删除。**

**代码证据（IdempotentAspect.java:56-66）：**
```java
try {
    return joinPoint.proceed();  // 正常返回直接 return，不执行后续逻辑
} catch (Throwable throwable) {
    // 只有异常时才会到这里
    if (idempotent.deleteKeyWhenException()) {
        idempotentRedisDAO.delete(key);
    }
    throw throwable;
}
```

**流程分析：**
```
正常返回路径：setIfAbsent 成功 → try { return proceed(); } → 方法结束
                                                         ↑
                                                   直接 return，跳过 catch 块
                                                   Redis key 保留，等待超时

异常路径：setIfAbsent 成功 → try { proceed() 抛异常 } → catch 块 → 根据 deleteKeyWhenException 决定是否删 key
```

**为什么不删除？**

注解源码注释（Idempotent.java:58-60）已明确说明：
```java
/**
 * 问题：为什么不搞 deleteWhenSuccess 执行成功时，需要删除 Key 呢？
 * 回答：这种情况下，本质上是分布式锁，推荐使用 @Lock4j 注解
 */
```

**设计意图：**

@Idempotent 的设计目标是**"防止重复请求"**，而不是**"分布式锁"**：

| 场景 | 设计目标 | Key 生命周期 |
|------|---------|-------------|
| 防重复请求（@Idempotent） | 同一请求（相同key）在超时窗口内只执行一次 | 成功后保留，等待超时过期 |
| 分布式锁（@Lock4j） | 同一时刻只允许一个线程执行 | 执行完成立即释放 |

**实际意义：**

假设超时时间为 1 秒：
- t=0：请求 A 执行成功，key 保留
- t=0.5 秒：用户重复点击（相同 key），请求 B 被拒绝（key 还在）
- t=1.1 秒：key 过期，新请求可以执行

这确保了用户在短时间内的重复操作（如双击按钮）不会导致业务重复执行。

---

### 4）业务方法抛异常时 deleteKeyWhenException 为 true 或 false 对重试窗口有什么影响

**注解定义（Idempotent.java:52-61）：**
```java
/**
 * 删除 Key，当发生异常时候
 *
 * 问题：为什么发生异常时，需要删除 Key 呢？
 * 回答：发生异常时，说明业务发生错误，此时需要删除 Key，避免下次请求无法正常执行。
 */
boolean deleteKeyWhenException() default true;
```

**两种情况对比：**

| 配置 | 异常后行为 | 重试窗口 | 适用场景 |
|------|-----------|---------|---------|
| `deleteKeyWhenException = true`（默认） | **立即删除 key** | 异常后**立即可重试** | 临时性故障（网络波动、数据库超时等），希望用户能立即重试 |
| `deleteKeyWhenException = false` | **保留 key 直到超时** | 必须等待**超时后才能重试** | 业务逻辑错误，需要排查后再重试，避免重复提交有问题的请求 |

**时间线对比（假设 timeout=5s，业务执行 2s 后抛异常）：**

**deleteKeyWhenException = true：**
```
t=0    : 请求到达 → setIfAbsent 成功 → 开始执行
t=2    : 业务抛异常 → 立即删除 key
t=2.1  : 用户重试 → setIfAbsent 成功（key 已被删）→ 可重新执行
       ↑
       异常后立即可重试
```

**deleteKeyWhenException = false：**
```
t=0    : 请求到达 → setIfAbsent 成功 → 开始执行
t=2    : 业务抛异常 → 不删除 key（保留）
t=2.1  : 用户重试 → setIfAbsent 失败（key 还在）→ 被拒绝
t=5    : key 超时过期
t=5.1  : 用户重试 → setIfAbsent 成功 → 可重新执行
       ↑
       必须等待 5 秒后才能重试
```

**参考设计：**

代码注释提到参考了**美团 GTIS** 的思路（美团的分布式互斥幂等方案 Cerberus），异常时删除 key 是为了让用户有机会重试失败的请求。

---

### 5）UserIdempotentKeyResolver 在未登录、login-user header 被网关剥离、绕过网关直连时可能有什么差异

**UserIdempotentKeyResolver 实现（UserIdempotentKeyResolver.java:19-26）：**
```java
@Override
public String resolver(JoinPoint joinPoint, Idempotent idempotent) {
    String methodName = joinPoint.getSignature().toString();
    String argsStr = StrUtils.joinMethodArgs(joinPoint);
    Long userId = WebFrameworkUtils.getLoginUserId();
    Integer userType = WebFrameworkUtils.getLoginUserType();
    return SecureUtil.md5(methodName + argsStr + userId + userType);
}
```

Key 组成：`MD5(方法名 + 参数 + userId + userType)`

**WebFrameworkUtils 获取逻辑（WebFrameworkUtils.java:91-132）：**
- `getLoginUserId()`：从 `request.getAttribute("login_user_id")` 获取
- `getLoginUserType()`：优先从 `request.getAttribute("login_user_type")` 获取，其次根据 URL 前缀推断

**三种场景分析：**

| 场景 | userId | userType | 影响 |
|------|--------|----------|------|
| **正常登录** | 有值（如 1001） | 有值（如 1=ADMIN, 2=MEMBER） | 不同用户的相同操作生成不同 key，互不干扰 |
| **未登录** | `null` | 可能 `null` 或根据 URL 推断 | 所有未登录用户共享同一个 key（如果参数相同），一个用户操作后，其他未登录用户短时间内无法执行相同操作 |
| **login-user header 被网关剥离** | `null`（网关没设置 attribute） | 可能 `null` 或根据 URL 推断 | 同上，所有用户被当作"同一人"处理 |
| **绕过网关直连** | `null`（无中间件设置 attribute） | 可能 `null` 或根据 URL 推断 | 同上，所有直连请求共享 key |

**详细差异：**

#### 场景 1：正常登录（网关正确传递用户信息）
```
网关 → 解析 Token → setAttribute("login_user_id", 1001)
                              setAttribute("login_user_type", 2)
                                                ↓
                                         服务接收到请求
                                                ↓
                                    userId=1001, userType=2
                                                ↓
                              Key = MD5(方法 + 参数 + "1001" + "2")
```

- 用户 A（1001）和用户 B（1002）的相同操作生成不同 key
- 各自独立，互不影响

#### 场景 2：未登录（无 Token 或 Token 无效）
```
无 Token → 网关不设置 login_user_id attribute
                                ↓
                     userId=null, userType 可能为 null
                                ↓
                Key = MD5(方法 + 参数 + "null" + "null")
```

- 所有未登录用户使用**相同 key**
- 用户 A 执行一次后，用户 B（未登录）在超时时间内无法执行相同操作
- 这是一个**潜在问题**：未登录场景下可能出现"一人操作，众人等待"

#### 场景 3：login-user header 被网关剥离
```
原始请求有 Token，但网关配置错误/过滤器异常
                                ↓
                  没有调用 setAttribute("login_user_id", ...)
                                ↓
                     userId=null, userType 可能为 null
                                ↓
                Key = MD5(方法 + 参数 + "null" + "null")
```

- 所有请求被当作"同一用户"处理
- 不同真实用户的操作互相干扰
- **安全隐患**：可能导致正常用户的请求被错误拒绝

#### 场景 4：绕过网关直连服务
```
直接请求服务端口（如 8080），不经过网关
                                ↓
                  没有中间件设置 login_user_id
                                ↓
                     userId=null, userType 可能为 null
                                ↓
                Key = MD5(方法 + 参数 + "null" + "null")
```

- 同场景 3，所有直连请求共享 key
- 这意味着**绕过网关的攻击者**可能更容易触发"重复请求"保护，或者干扰正常用户

**userType 的特殊逻辑：**

`getLoginUserType()` 有兜底逻辑（WebFrameworkUtils.java:109-121）：
```java
// 1. 优先从 Attribute 获取
Integer userType = (Integer) request.getAttribute(REQUEST_ATTRIBUTE_LOGIN_USER_TYPE);
if (userType != null) {
    return userType;
}
// 2. 其次根据 URL 前缀推断
if (request.getServletPath().startsWith(properties.getAdminApi().getPrefix())) {
    return UserTypeEnum.ADMIN.getValue();  // 1
}
if (request.getServletPath().startsWith(properties.getAppApi().getPrefix())) {
    return UserTypeEnum.MEMBER.getValue(); // 2
}
return null;
```

所以即使 userId 为 null，userType 可能仍有值（根据 `/admin-api/` 或 `/app-api/` 前缀）。

---

### 6）用两个并发请求时间线推导方法执行次数、Redis key 状态和异常返回

**假设条件：**
- 注解配置：`@Idempotent(timeout = 5, timeUnit = SECONDS, deleteKeyWhenException = true)`
- 业务方法执行时间：3 秒（假设方法内有 `Thread.sleep(3000)` 或实际业务耗时）
- 两个请求使用完全相同的参数（生成相同 key）
- 请求 A 在 `t=0` 到达，请求 B 在 `t=1秒` 到达

---

#### 时间线推导

```
t=0s  ────────────────────────────────────────────────────────── t=5s  ──→
  │                                                                 │
  ├── 请求 A 到达
  │      │
  │      ├── 解析 key: "abc123"
  │      │
  │      ├── setIfAbsent("idempotent:abc123", "", 5s)
  │      │      │
  │      │      └── Redis 中无此 key → 返回 true
  │      │
  │      ├── 开始执行业务方法 (耗时 3s)
  │      │
  │      │  t=1s ───────┐
  │      │      │       │
  │      │      ├── 请求 B 到达
  │      │      │      │
  │      │      ├── 解析 key: "abc123" (相同)
  │      │      │      │
  │      │      ├── setIfAbsent("idempotent:abc123", "", 5s)
  │      │      │      │
  │      │      │      └── Redis 中已有此 key → 返回 false
  │      │      │      │
  │      │      ├── 记录日志: "存在重复请求"
  │      │      │
  │      │      └── 抛出 ServiceException: "重复请求，请稍后重试"
  │      │             │
  │      │             └── 请求 B 结束 ❌
  │      │
  │      ├── t=3s 业务方法执行完成
  │      │
  │      ├── 正常返回（不删除 Redis key）
  │      │
  │      └── 请求 A 结束 ✅
  │
  ├── Redis key "idempotent:abc123" 持续存在
  │       (从 t=0 到 t=5)
  │
  ├── t=3s ~ t=5s 期间:
  │       任何新请求都会被拒绝
  │
  └── t=5s: Redis key 超时自动删除
          新请求可以重新执行
```

---

#### 结果汇总

| 指标 | 请求 A | 请求 B |
|------|--------|--------|
| 到达时间 | t=0s | t=1s |
| setIfAbsent 结果 | ✅ true | ❌ false |
| 业务方法执行次数 | 1 次 | 0 次 |
| Redis key 状态变化 | t=0 创建，t=5 过期 | 无变化（key 已存在） |
| 返回结果 | 业务正常返回 | 抛出 ServiceException |
| 异常类型 | 无 | `ServiceException`（重复请求） |
| 用户感知 | 操作成功 | 看到"重复请求，请稍后重试"提示 |

---

#### Redis key 状态完整时间线

| 时间点 | Key 状态 | 说明 |
|--------|---------|------|
| t<0 | 不存在 | 初始状态 |
| t=0 | 已创建（TTL=5s） | 请求 A setIfAbsent 成功 |
| 0<t<5 | 存在（TTL 递减） | 期间任何请求都被拒绝 |
| t=3 | 仍存在（TTL=2s） | 请求 A 完成但不删除 key |
| t=5 | 已过期删除 | 超时自动删除 |
| t>5 | 不存在 | 新请求可以执行 |

---

#### 异常场景变体（业务方法抛异常）

如果请求 A 在 t=2s 时抛异常（`deleteKeyWhenException=true`）：

```
t=0  : 请求 A 到达 → setIfAbsent 成功 → 开始执行
t=2  : 业务抛异常 → 立即删除 key
t=2.1: 请求 B (重试) → setIfAbsent 成功 → 可重新执行
```

- 请求 A：执行 1 次，抛异常
- 请求 B（t=1s 的那个）：被拒绝（因为 t=1 时 key 还在）
- 请求 C（t=2.1s 重试）：可以执行

---

## 三、完整执行流程图

```
@Idempotent(timeout=5, keyResolver=UserIdempotentKeyResolver.class, deleteKeyWhenException=true)
                          │
                          ▼
                 ┌─────────────────┐
                 │ IdempotentAspect │
                 │  aroundPointCut  │
                 └────────┬────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐
   │ 解析 Key    │  │ 获取配置    │  │ 获取 Resolver│
   │ 解析器      │  │ (timeout,   │  │ 从 Map      │
   │ User...    │  │  message,   │  │ 匹配        │
   └─────┬──────┘  │  delete...) │  └─────┬──────┘
         │         └────────────┘        │
         └───────────────┬───────────────┘
                         ▼
                 ┌─────────────────┐
                 │ 解析唯一 Key     │
                 │ MD5(方法+参数+    │
                 │ userId+userType) │
                 └────────┬────────┘
                         ▼
                 ┌─────────────────┐
                 │ Redis setIfAbsent│
                 │ SETNX + 过期时间  │
                 └────────┬────────┘
              ┌───────────┴───────────┐
              ▼                       ▼
         ┌─────────┐            ┌───────────┐
         │  成功    │            │   失败     │
         │(抢到锁) │            │(重复请求)  │
         └────┬────┘            └─────┬─────┘
              │                       │
              ▼                       ▼
     ┌───────────────┐       ┌──────────────────┐
     │ 执行业务方法    │       │ 记录日志          │
     │ joinPoint.    │       │ 抛出 Service-    │
     │ proceed()     │       │ Exception        │
     └───────┬───────┘       └────────┬─────────┘
             │                        │
    ┌────────┴────────┐              结束
    ▼                 ▼
┌─────────┐      ┌───────────┐
│ 正常返回 │      │  抛异常    │
│ 不删 Key │      │ deleteKey │
│ 等待超时 │      │ WhenExcept│
└────┬────┘      │ ion=true? │
     │           └─────┬─────┘
     │            ┌────┴────┐
     │            ▼         ▼
     │        ┌──────┐  ┌───────┐
     │        │ 是   │  │  否   │
     │        │删Key │  │保留   │
     │        └──┬───┘  └───┬───┘
     │           │          │
     └───────────┴──────────┘
                 │
                结束
```

---

## 四、关键设计总结

| 设计点 | 决策 | 理由 |
|--------|------|------|
| Key 存储 | Redis SETNX + TTL | 原子性保证，防止并发竞争 |
| 成功后不删 Key | 等待超时过期 | 防止用户短时间内重复操作（双击等） |
| 异常后默认删 Key | `deleteKeyWhenException=true` | 允许用户重试临时性失败的请求 |
| Key 解析器选择 | Map<Class, Instance> | Spring 自动发现，灵活扩展 |
| 多 Key 解析器 | Default/User/Expression | 适应不同粒度的幂等需求 |

@Idempotent 的核心思想是：**在超时窗口内，对相同 key 的请求只允许第一次成功执行，后续请求直接拒绝。** 这与分布式锁的"执行完立即释放"有本质区别。
